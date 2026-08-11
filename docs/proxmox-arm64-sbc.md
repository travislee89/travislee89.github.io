# 在不支持 UEFI 的 ARM64 SBC 上安装 Proxmox VE

Proxmox VE 官方已经提供 ARM64 支持，但官方的安装流程是基于 UEFI + ACPI 设计的：它会安装 `proxmox-kernel`（Proxmox 自己编译的内核）和 `pve-firmware`，并假定硬件走标准的 UEFI 启动流程。

问题是，绝大多数 ARM64 SBC（比如各类 Rockchip、Allwinner 方案的板子）既不支持 UEFI，也不支持 ACPI，只能用厂商定制的内核（比如基于 Armbian 的内核）通过 u-boot 之类的方式启动。如果照官方流程装 `proxmox-kernel`，SBC 大概率直接起不来。

解决思路是：**保留 SBC 厂商自带的内核，用 `equivs` 做两个空的“占位包”去满足 `pve-manager` 对 `proxmox-default-kernel` 和 `pve-firmware` 的依赖，然后把这两个真实包锁定为禁止安装**，这样既能装上完整的 Proxmox VE 管理套件，又不会覆盖掉 SBC 能正常启动所依赖的内核。

下面是完整流程。

## 1. 设置主机名、IP 和网络接口变量

请根据自己的需求修改

```bash
hostname=pve
ip=192.168.1.64/24
gateway=192.168.1.1
interfance=eth0

echo "${hostname}" | sudo tee /etc/hostname
echo "${ip} ${hostname}.local ${hostname}" | sudo tee -a /etc/hosts
```

按自己的网络环境改 `hostname`、`ip`、`gateway`、`interfance`（网卡名）即可。

## 2. 添加 Proxmox APT 源

官方仓库本身就带 arm64 架构的包，只需要在源里显式声明 `Architectures: arm64`：

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

## 3. 网络栈切换为 ifupdown2

Proxmox 的网络管理（网桥、VLAN、SDN 等）依赖传统的 `ifupdown` 体系，跟 NetworkManager、systemd-networkd 冲突，需要提前禁用后者：

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

## 4. 配置 `/etc/network/interfaces`，建立网桥

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

`vmbr0` 是 Proxmox 里虚拟机、容器默认挂载的网桥，`bridge-vlan-aware yes` 开启后可以直接在网桥上跑多个 VLAN。改完配置文件后重启`sudo reboot`，让新的网络接管生效。

## 5. 用 equivs 伪造内核和固件依赖包

重启回来后，先装 `equivs`（用来制作空的占位 deb 包的工具）：

```bash
sudo apt install equivs -y
```

### 5.1 伪造 `proxmox-default-kernel`

官方 `pve-manager` 强制依赖 `proxmox-default-kernel`，这个包会拉入真正的 Proxmox 内核，并假设走 UEFI 启动。SBC 不需要也用不了这个内核，所以要做一个空包去“顶替”这个依赖：

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

这个函数做了四件事：

1. 确认 `equivs` 已安装；
2. 从当前 APT 仓库里查出所有实际存在的 `proxmox-kernel-*` 包名，作为 `Provides` 字段填进 dummy 包，这样任何声明依赖具体某个 `proxmox-kernel-x.y` 版本的包也会被这个空包“满足”；
3. 用 `equivs-build` 打出一个版本号为 `99.0.0` 的假 `proxmox-default-kernel` 包并安装——版本号故意给得很大，让 APT 永远认为它是“最新版”，不会被真正的内核包替换掉；
4. 写入 APT pin 规则，把所有 `proxmox-kernel-*` 和 `proxmox-headers-*` 的 `Pin-Priority` 设为 `-1`（禁止安装），并对 `proxmox-default-kernel` 执行 `apt-mark hold`，防止后续 `apt upgrade` 时把 SBC 自身能用的内核换掉。

### 5.2 伪造 `pve-firmware`

同理，`pve-firmware` 里打包的是常见的网卡/显卡固件，SBC 不需要，也用不上：

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

逻辑和内核包一样：打一个版本号 `99.0.0` 的空包声明 `Provides: pve-firmware`，装上后 `apt-mark hold`，锁死版本,不让真正的 `pve-firmware` 覆盖进来。

### 5.3 静默安装postfix

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

### 5.4 执行伪造并安装 Proxmox VE

```bash
mk_pve_firmware_dummy
mk_pve_kernel_dummy
setup_postfix_local
sudo apt install -y proxmox-ve pve-manager
```

两个空包装好、依赖满足之后，`apt install proxmox-ve pve-manager` 就能顺利跑完，不会再去拉取那些依赖 UEFI 启动的真实内核和固件包。

## 6. 修复根 CA 证书（如遇 pveproxy/pvedaemon 无法启动）

部分 SBC 环境下首次安装后，`/etc/pve/priv/pve-root-ca.key` 或 `/etc/pve/pve-root-ca.pem` 可能是空文件或缺失，导致 Web 管理界面对应的服务起不来。可以检测并自动修复：

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

这个函数先检查证书和私钥文件是否存在且非空；如果异常，就删掉损坏的文件，用 `pvecm updatecerts -f` 重新生成节点证书，再重启 `pveproxy`（Web 界面代理）和 `pvedaemon`（核心管理进程），最后打印两个服务的状态供确认。

## 7. 移除企业版订阅源

```bash
sudo rm -f /etc/apt/sources.list.d/pve-enterprise.sources
```

不移除的话，没有商业订阅的机器每次 `apt update` 都会因为访问企业版源报错或弹出订阅提示。

## 小结

整套方案能在不支持 UEFI/ACPI 的 ARM64 SBC 上装起官方 Proxmox VE，核心不是绕过官方源，而是**绕过官方安装流程里"必须装真正的 proxmox-kernel 和 pve-firmware"这一步**：用 `equivs` 打两个版本号故意设得很高（`99.0.0`）的空包去满足 `pve-manager` 的依赖声明，再用 APT pin 和 `apt-mark hold` 双重锁死，防止升级时被真实包顶替，从而保留 SBC 厂商自带、真正能引导启动的内核。装完之后网络、Web 管理界面、证书等和标准 x86_64 安装基本一致。
