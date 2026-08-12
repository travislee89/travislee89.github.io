# travislee89.github.io

我的个人博客（GitHub Pages）。

## 目录结构

```
docs/                     # GitHub Pages 源目录（Settings → Pages → Source: /docs）
├── _config.yml           # 站点配置（标题、主题、permalink、插件）
├── Gemfile               # 本地构建依赖（github-pages gem）
├── index.html            # 首页：自动列出站点文章
├── about.md              # 关于页
├── _layouts/             # 页面布局（default / home / post）
├── _posts/               # 博客文章（文件名带日期，YYYY-MM-DD-title.md）
└── assets/css/main.css   # 站点样式
```

## 文章命名与互链

- 文章放在 `docs/_posts/`，文件名格式 `YYYY-MM-DD-标题.md`，带 YAML front matter。
- 中英双语采用「两篇独立 post + 互链」：每篇用 `lang` 标注语言，用 `translation`
  字段指向另一语言版本，`_layouts/post.html` 会自动渲染切换链接。

## 本地构建

```bash
cd docs
bundle install      # 依赖 github-pages gem，与线上一致
bundle exec jekyll serve   # http://localhost:4000
```

变更推送到 `main` 分支后，GitHub Pages 会自动以 `/docs` 为源构建上线。
