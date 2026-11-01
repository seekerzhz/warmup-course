# 部署说明

这份文档只讲「怎么把站点发出去」，不讲课程内容。

## 结构

- 源码在 `main` 分支，`_book/` 不进仓库（`.gitignore` 已忽略）
- 构建由 GitHub Actions 完成：push 到 `main` 自动触发，也可以在 Actions 页面手动 `workflow_dispatch`
- 构建产物同时走两条路：
  1. **Actions 产物部署**（主通道，`actions/deploy-pages`）：不需要 `gh-pages` 分支，和 `lsm` 仓库的做法一致
  2. **`gh-pages` 分支**（备份通道，`peaceiris/actions-gh-pages`）：产物镜像到这个分支，方便切回分支部署
- 站点入口：`https://docs.hanzel.top/warmup-course/`

## 为什么没有 CNAME

`docs.hanzel.top` 绑在 `seekerzhz.github.io`（用户站点）上。GitHub Pages 的规则是：**自定义域只挂在用户站点上，项目仓库自动作为子路径**。所以本项目仓库不加 `CNAME` 文件，否则两个仓库抢同一个域会互相打架。

## 如果 Pages 没自动启用

正常情况 `actions/configure-pages` 的 `enablement: true` 会自动把 Pages 打开，发布源设为 `GitHub Actions`。如果仓库策略不允许自动启用，需要用户手动做一次：

Settings → Pages → Build and deployment → Source 选 `GitHub Actions`（或选 `Deploy from a branch` + `gh-pages` / `root`）。

只需要做一次，之后每次 push 都会自动更新。

## 子路径资源

HonKit 默认输出**相对路径**（`gitbook/style.css`），所以放在 `/warmup-course/` 子路径下天然可用，不需要额外处理。

workflow 里的 `sed` 只是保险：万一某个 HonKit 版本或插件输出了绝对路径 `/gitbook/...`，这一步会把它改写成 `/warmup-course/gitbook/...`。

## 想换成 `/warmup/` 短路径

1. 改仓库名：`gh repo rename warmup -R seekerzhz/warmup-course`
2. 把 `.github/workflows/deploy.yml` 里的 `warmup-course` 全部替换成 `warmup`
3. 重新 push，触发一次构建

站点会变成 `https://docs.hanzel.top/warmup/`，旧路径失效。
