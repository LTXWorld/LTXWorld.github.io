# 博客系统搭建分析

## 1. 技术栈与整体架构
- 静态站点生成器：项目使用 Hugo（配置文件为 `hugo.yaml`）构建博客内容，结合其模板语言生成最终 HTML。
- 主题：通过 Git 子模块引入 `PaperMod` 主题（`.gitmodules` 指定 `https://github.com/adityatelange/hugo-PaperMod.git`），并在 `hugo.yaml` 中通过 `theme: PaperMod` 使用。
- 部署：`baseURL` 指向 GitHub Pages 提供的 `https://LTXWorld.github.io/`，仓库名与 Pages 命名空间一致（`LTXWorld/LTXWorld.github.io`），根目录的 `CNAME` 指定自定义域名 `www.bfsmlt.top`，因此最终托管在 GitHub Pages 上。
- 评论与交互：通过自定义模板集成 giscus（GitHub Discussions 驱动的评论系统），搜索使用主题内置的 Fuse.js（`outputs.home` 包含 `JSON` 以便客户端搜索）。
- 内容语言与本地化：`languageCode: zh-cn`、`hasCJKLanguage: true` 与 `defaultContentLanguage: zh` 以优化中文排版和默认语言体验。

## 2. 搭建步骤概述
1. **准备开发环境**：安装 Hugo Extended 版本（包含对 SCSS 管道的支持）以及 Git；若计划在本地预览，需要可访问外网以加载主题子模块和 giscus。
2. **获取主题**：首次克隆仓库后执行 `git submodule update --init --recursive` 下载 `themes/PaperMod` 主题；后续更新主题时可使用 `git submodule update --remote`。
3. **站点配置**：在 `hugo.yaml` 中维护站点标题、导航菜单、分页、社交链接、暗色模式默认值以及 giscus 配置（`repo`、`categoryId` 等）。
4. **撰写内容**：
   - `archetypes/default.md` 定义文章的默认 Front Matter，可使用 `hugo new posts/<slug>.md` 创建新稿件。
   - 正文位于 `content/posts/`（根据需求创建子目录或草稿文件），避免直接修改 `public/` 中的构建结果。
5. **主题定制**：
   - `layouts/partials/extend_head.html` 追加 Google Fonts 与 MathJax；`layouts/partials/mathjax.html` 使用国内 CDN (`cdn.bootcss.com`) 提供的 MathJax 2.x 并处理 `$...$` 等语法；`layouts/partials/comments.html` 动态注入 giscus 脚本并在主题切换时更新评论区配色。
   - `assets/css/extended/` 目录覆写 PaperMod SCSS 变量（`theme-vars-override.css`）、滚动条、代码块字体等；`syntax.css` 覆写 Chroma 代码高亮配色，确保与暗色主题一致。
   - `layouts/_default/about.html` 提供关于页模板，保持与主题主体一致。
6. **本地开发**：运行 `hugo server -D` 启动调试服务器，`-D` 允许预览草稿文章；必要时更换 `baseURL` 为 `http://localhost:1313/` 以避免资源路径问题。
7. **构建发布**：执行 `hugo` 输出静态文件到 `public/` 目录；该目录目前包含已经构建好的页面，部署到 GitHub Pages 时可通过 GitHub Actions 或 `gh-pages` 分支公开。
8. **部署到 GitHub Pages**：推送仓库到 `main`（或 `master`）分支，同时启用 Pages；若采用 `public/` 目录作为发布源，可通过 GitHub Pages 的「使用 docs/」选项或单独仓库存放构建结果。

## 3. 关键技术点
- **Hugo 配置能力**：通过 `enableRobotsTXT`、`buildDrafts` 等参数控制构建行为；`enableInlineShortcodes` 支持在 Markdown 中混合短代码，`hasCJKLanguage` 解决中文计数和排版。
- **多语言与导航**：`languages.zh` 节维护中文站点的菜单、分类（`taxonomies`）等信息，确保 Hugo 在多语言模式下正确生成页面。
- **PaperMod 功能扩展**：利用 `params` 启用 `ShowReadingTime`、`ShowCodeCopyButtons`、`ShowAllPagesInArchive` 等主题特性，同时通过 `homeInfoParams` 自定义首页简介与社交链接。
- **评论系统 giscus**：自定义部分负责在 DOMContentLoaded 之后注入 `giscus.app` 脚本，并监听主题切换按钮（`#theme-toggle`/`#theme-toggle-float`）来动态更新评论主题，保证暗/亮主题一致性。
- **数学公式支持**：`mathjax.html` 加载 MathJax 并针对 Markdown 代码块应用 `has-jax` class，使公式与代码块样式兼容。
- **样式覆写**：`assets/css/extended/*` 与 `syntax.css` 共同定义字体（JetBrains Mono）、滚动条样式及代码高亮色板，以匹配博客的视觉风格。

## 4. 目录结构速览
- `archetypes/`：Front Matter 模板，保证新文章字段一致。
- `assets/css/extended/`：PaperMod 主题的定制化样式覆写，构建时由 Hugo 管道注入。
- `layouts/`：额外模板片段（`partials/`）与页面模板（`_default/about.html`）用于增强主题功能。
- `static/`：直接复制到输出目录的静态资源，如 `favicon.png` 与图片。
- `syntax.css`：直接提供给主题使用的 Chroma 配色文件。
- `themes/PaperMod`：Git 子模块位置，需在本地拉取以获得主题源码。
- `public/`：`hugo` 命令生成的静态页面成果，用于部署；开发时不应手工修改。

## 5. 部署与维护建议
- 在本地或 CI 流程中确保 Git 子模块已更新，否则构建将缺失主题资源。
- 若启用 GitHub Actions，可在 CI 中执行 `hugo --minify` 并将 `public/` 发布到 `gh-pages` 分支或 `docs/` 目录。
- 保持 `CNAME` 文件与 GitHub Pages 设置一致，避免自定义域名失效。
- 定期检查 giscus 配置项（`repoId`、`categoryId` 等）是否因仓库迁移或权限变动而需要更新。
- 如果需要升级到 MathJax 3 或替换 CDN，需同步更新 `layouts/partials/mathjax.html` 的语法配置。

## 6. 后续扩展思路
- 为保证搜索索引实时更新，可在每次构建后检查 `public/index.json`（PaperMod 生成的搜索索引）是否包含近期文章。
- 通过 `params.assets` 可以为不同平台定义更多图标（如 PWA）；也可以在 `extend_head.html` 中加入统计脚本或自定义 CSS。
- 若要引入英文界面，可在 `languages` 中新增 `en` 节并为菜单、分类提供对应翻译。

以上内容概括了仓库中体现的配置、技术栈以及搭建流程，可按需调整以适应后续的维护与扩展。
