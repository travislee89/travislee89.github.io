---
layout: post
title: Installing Proxmox VE on ARM64 SBCs Without UEFI Support
date: 2026-08-12
lang: en
translation: /proxmox/2026/08/12/proxmox-arm64-sbc/
categories: proxmox
tags: [proxmox, arm64, sbc, linux]
---

# Installing Proxmox VE on ARM64 SBCs Without UEFI Support

Proxmox VE now officially provides ARM64 support, but the official installation process is designed around UEFI + ACPI. It installs `proxmox-kernel` (the kernel built by Proxmox) and `pve-firmware`, and assumes that the hardware follows a standard UEFI boot process.

The problem is that most ARM64 SBCs, including many Rockchip- and Allwinner-based boards, do not support UEFI or ACPI. They can only boot with a vendor-specific kernel, such as an Armbian kernel, through a bootloader like U-Boot. If you install `proxmox-kernel` using the official process, the SBC will most likely fail to boot.

The solution is to **keep the SBC vendor's existing kernel, use `equivs` to create two empty dummy packages that satisfy `pve-manager`'s dependencies on `proxmox-default-kernel` and `pve-firmware`, and then block the real packages from being installed**. This allows you to install the full Proxmox VE management stack without replacing the kernel that the SBC actually needs in order to boot.

The complete procedure is below.

## 1. Set the hostname, IP address, and network interface variables

Adjust these values to match your environment:

```bash
hostname=pve
ip=192.168.1.64/24
gateway=192.168.1.1
interfance=eth0

echo "${hostname}" | sudo tee /etc/hostname
echo "${ip} ${hostname}.local ${hostname}" | sudo tee -a /etc/hosts
```

Change `hostname`, `ip`, `gateway`, and `interfance` (the network interface name) according to your own network setup.

## 2. Add the Proxmox APT repository

The official repository already includes arm64 packages. You only need to explicitly declare `Architectures: arm64` in the source configuration:

```bash
sudo wget https://enterprise.proxmox.com/debian/proxmox-release-trixie.gpg \
     -O /etc/apt/trusted.gpg.d/proxmox-release-trixie.gpg

echo "Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Architectures: arm64
Signed-By: /etc/apt/trusted.gpg.d/proxmox-release-trixie.gpg
" | \
sudo tee /etc/apt/sources.list.d/pve-no-subscription.sources

sudo apt update
sudo apt upgrade -y
```

## 3. Switch the network stack to ifupdown2

Proxmox network management, including bridges, VLANs, and SDN, relies on the traditional `ifupdown` stack. It conflicts with NetworkManager and systemd-networkd, so those services should be disabled first:

```bash
sudo systemctl stop    NetworkManager
sudo systemctl disable NetworkManager
sudo systemctl mask    NetworkManager

sudo apt install ifupdown2 resolvconf cloud-image-utils -y
sudo systemctl enable  networking
sudo systemctl disable systemd-networkd
sudo systemctl disable systemd-networkd.socket
sudo apt remove -y netplan.io && sudo apt autoremove -y
```

## 4. Configure `/etc/network/interfaces` and create a bridge

```bash
echo "
auto lo
iface lo inet loopback

iface ${interfance} inet manual

auto vmbr0
iface vmbr0 inet static
    address ${ip}
    gateway ${gateway}
    dns-nameservers ${gateway}
    dns-search local
    bridge-ports ${interfance}
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

source /etc/network/interfaces.d/*

" | sudo tee /etc/network/interfaces;
```

`vmbr0` is the default bridge used by virtual machines and containers in Proxmox. With `bridge-vlan-aware yes` enabled, multiple VLANs can be carried directly over the bridge. After updating the configuration, reboot with `sudo reboot` so that the new network configuration takes over.

## 5. Use equivs to create dummy kernel and firmware dependency packages

After the system comes back up, install `equivs`, which is used to build empty placeholder `.deb` packages:

```bash
sudo apt install equivs -y
```

### 5.1 Create a dummy `proxmox-default-kernel` package

The official `pve-manager` package has a hard dependency on `proxmox-default-kernel`. That package normally pulls in the actual Proxmox kernel and assumes a UEFI-based boot environment. An SBC does not need, and usually cannot use, that kernel, so we create an empty package that satisfies the dependency instead:

```bash
mk_pve_kernel_dummy() {
    echo "==> [1/4] Checking dependency: equivs..."
    if ! command -v equivs-build &>/dev/null; then
        echo "    'equivs' not found. Installing automatically..."
        sudo apt-get update -qq && sudo apt-get install -y -qq equivs
    fi

    local workdir
    workdir=$(mktemp -d /tmp/pve-kernel-dummy.XXXXXX)
    trap 'rm -rf "$workdir"' RETURN EXIT

    echo "==> [2/4] Querying Proxmox kernel packages from APT repository..."
    local pve_kernels
    pve_kernels=$(apt-cache search "^proxmox-kernel-[0-9]" | awk '{print $1}' | tr '\n' ',' | sed 's/,$//')

    if [ -z "$pve_kernels" ]; then
        echo "    [!] Warning: No proxmox-kernel package found. Falling back to default virtual dependencies."
        pve_kernels="proxmox-kernel-7.0"
    else
        echo "    Retrieved kernel dependency list: ${pve_kernels}"
    fi

    echo "==> [3/4] Building dummy proxmox-default-kernel package..."
    cat > "${workdir}/proxmox-kernel-dummy" <<EOF
Section: kernel
Priority: optional
Standards-Version: 3.9.2

Package: proxmox-default-kernel
Version: 99.0.0
Provides: ${pve_kernels}, proxmox-headers-7.0
Maintainer: Local Dummy <root@localhost>
Architecture: all
Description: Dummy package for Proxmox kernel on Armbian
EOF

    (
        cd "$workdir"
        equivs-build proxmox-kernel-dummy >/dev/null
        sudo dpkg -i proxmox-default-kernel_99.0.0_all.deb
    )

    echo "==> [4/4] Configuring APT pinning rules and locking version..."
    sudo tee /etc/apt/preferences.d/pve-kernel-pin >/dev/null <<'EOF'
Package: proxmox-kernel-helper
Pin: release *
Pin-Priority: 500

Package: /^proxmox-kernel-[0-9].*/
Pin: release *
Pin-Priority: -1

Package: proxmox-headers-*
Pin: release *
Pin-Priority: -1
EOF

    sudo apt-mark hold proxmox-default-kernel >/dev/null
    echo "✓ Proxmox kernel dummy package successfully created and locked!"
}
```

This function does four things:

1. Checks that `equivs` is installed.
2. Queries the current APT repository for all available `proxmox-kernel-*` package names and adds them to the dummy package's `Provides` field. This allows the dummy package to satisfy dependencies that explicitly reference a specific `proxmox-kernel-x.y` package.
3. Builds and installs a fake `proxmox-default-kernel` package with version `99.0.0`. The intentionally high version number makes APT treat it as newer than the real package, preventing it from being replaced automatically.
4. Creates APT pinning rules that assign a `Pin-Priority` of `-1` to all `proxmox-kernel-*` and `proxmox-headers-*` packages, effectively blocking them from being installed. It also applies `apt-mark hold` to `proxmox-default-kernel` so that a later `apt upgrade` cannot replace the SBC's working kernel with a Proxmox kernel.

### 5.2 Create a dummy `pve-firmware` package

The same idea applies to `pve-firmware`. It contains firmware for common network and graphics hardware, which is generally not needed or useful on these SBCs:

```bash
mk_pve_firmware_dummy() {
    echo "==> [1/3] Checking dependency: equivs..."
    if ! command -v equivs-build &>/dev/null; then
        echo "    'equivs' not found. Installing automatically..."
        sudo apt update -qq && sudo apt install -y -qq equivs
    fi

    local workdir
    workdir=$(mktemp -d /tmp/pve-firmware-dummy.XXXXXX)
    trap 'rm -rf "$workdir"' RETURN EXIT

    echo "==> [2/3] Building dummy pve-firmware package..."
    cat > "${workdir}/pve-firmware-dummy" <<EOF
Section: misc
Priority: optional
Standards-Version: 3.9.2

Package: pve-firmware
Version: 99.0.0
Provides: pve-firmware
Maintainer: Local Dummy <root@localhost>
Architecture: all
Description: Dummy package for PVE firmware on Armbian
EOF

    (
        cd "$workdir"
        equivs-build pve-firmware-dummy >/dev/null
        sudo dpkg -i pve-firmware_99.0.0_all.deb
    )

    echo "==> [3/3] Locking pve-firmware version..."
    sudo apt-mark hold pve-firmware >/dev/null
    echo "Proxmox firmware dummy package successfully created and locked!"
}
```

The logic is the same as for the kernel package: create an empty package with version `99.0.0`, declare `Provides: pve-firmware`, install it, and then lock it with `apt-mark hold` so that the real `pve-firmware` package cannot replace it.

### 5.3 Install Postfix silently

```bash
setup_postfix_local() {
    echo "==> Setting up Postfix automatically as 'Local only'..."

    # Install debconf-utils if debconf-set-selections is missing
    if ! command -v debconf-set-selections &>/dev/null; then
        sudo apt install -y -qq debconf-utils
    fi

    local mail_domain
    mail_domain=$(hostname -f 2>/dev/null || hostname)

    # Inject debconf values for unattended installation
    sudo debconf-set-selections <<EOF
postfix postfix/main_mailer_type select Local only
postfix postfix/mailname string ${mail_domain}
EOF

    # Run installation non-interactively
    DEBIAN_FRONTEND=noninteractive sudo apt-get install -y postfix

    echo "Postfix successfully installed as Local only!"
}
```

### 5.4 Create the dummy packages and install Proxmox VE

```bash
mk_pve_firmware_dummy
mk_pve_kernel_dummy
setup_postfix_local
sudo apt install -y proxmox-ve pve-manager
```

Once both dummy packages are installed and the dependencies are satisfied, `apt install proxmox-ve pve-manager` can complete successfully without pulling in the real kernel and firmware packages that assume a UEFI-based boot process.

## 6. Repair the root CA certificate if `pveproxy` or `pvedaemon` fails to start

On some SBC environments, `/etc/pve/priv/pve-root-ca.key` or `/etc/pve/pve-root-ca.pem` may be missing or created as an empty file after the initial installation. This can prevent the services behind the Web management interface from starting. The following function detects and repairs the problem automatically:

```bash
fix_pve_certs() {
    local key_file="/etc/pve/priv/pve-root-ca.key"
    local cert_file="/etc/pve/pve-root-ca.pem"
    local need_repair=false

    # Check if either file does not exist or has a size of 0
    if [ ! -s "$key_file" ] || [ ! -s "$cert_file" ]; then
        need_repair=true
    fi

    if [ "$need_repair" = true ]; then
        echo "==> Missing or empty CA certificate/key detected. Repairing..."

        echo "    Removing corrupted CA files..."
        sudo rm -f "$key_file" "$cert_file"

        echo "    Regenerating Proxmox node certificates..."
        sudo pvecm updatecerts -f

        echo "    Restarting pveproxy and pvedaemon..."
        sudo systemctl restart pveproxy pvedaemon
    else
        echo "==> CA certificate and key are valid. No repair needed."
    fi

    echo "==> Checking service statuses..."
    sudo systemctl status pveproxy pvedaemon --no-pager
}

fix_pve_certs
```

The function first checks whether the certificate and private-key files exist and are non-empty. If either file is missing or invalid, it removes the broken files, regenerates the node certificates with `pvecm updatecerts -f`, restarts `pveproxy` (the Web interface proxy) and `pvedaemon` (the core management daemon), and finally prints the status of both services for verification.

## 7. Remove the enterprise subscription repository

```bash
sudo rm -f /etc/apt/sources.list.d/pve-enterprise.sources
```

If you do not remove it, systems without a commercial subscription may see errors during every `apt update` when accessing the enterprise repository, or receive subscription-related warnings.

## Summary

This approach makes it possible to install the official Proxmox VE packages on ARM64 SBCs that do not support UEFI or ACPI. The key is not to bypass the official repository, but to **bypass the part of the official installation process that requires the real `proxmox-kernel` and `pve-firmware` packages**.

Two dummy packages with deliberately high version numbers (`99.0.0`) are created using `equivs` to satisfy `pve-manager`'s dependency requirements. APT pinning and `apt-mark hold` are then used together to prevent the real packages from replacing them during upgrades. This preserves the SBC vendor's own kernel, which is the kernel that can actually boot the board.

Once installed, networking, the Web management interface, certificates, and the rest of the Proxmox environment work much like they do on a standard x86_64 installation.
