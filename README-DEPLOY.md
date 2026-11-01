# 部署说明

这份文档只讲「怎么把站点发出去」，不讲课程内容。

## 结构

- 源码在 `main` 分支，`_book/` 不进仓库（`.gitignore` 已忽略，`.bookignore` 让它也不进产物）
- 构建由 GitHub Actions 完成：push 到 `main` 自动触发，也可以在 Actions 页面手动 `workflow_dispatch`
- 构建产物走两条路：
  1. **`gh-pages` 分支**（主通道，`peaceiris/actions-gh-pages`）：产物推到这个分支
  2. **Actions 产物部署**（备用通道，`actions/deploy-pages`）：Pages 发布源选 `GitHub Actions` 时走这条
- 两条路产物完全一样，选哪个发布源都能用
- 站点入口：`https://docs.hanzel.top/warmup-course/`

## 一次性手动操作：启用 Pages

`GITHUB_TOKEN` 没有创建 Pages 站点的权限（报 `Resource not accessible by integration`），所以**第一次必须手动开一次**：

1. 打开 https://github.com/seekerzhz/warmup-course/settings/pages
2. **Source** 选 `Deploy from a branch`
3. **Branch** 选 `gh-pages`，目录选 `/ (root)`
4. Save

之后每次 push 都会自动更新，不用再动设置。

（如果更喜欢让 Actions 直接发布：同样的地方 Source 选 `GitHub Actions`。两条路都通，选一个就行。）

## 为什么没有 CNAME

`docs.hanzel.top` 绑在 `seekerzhz.github.io`（用户站点）上。GitHub Pages 的规则是：**自定义域只挂在用户站点上，项目仓库自动作为子路径**。所以本项目仓库**不加** `CNAME` 文件 —— 加了会变成「项目仓库抢自定义域」，和用户站点打架，`/lsm/` 也可能被影响。

这也是 workflow 里没有 `cname:` 那一行的原因。

## 子路径资源

HonKit 默认输出**相对路径**（`gitbook/style.css`），放在 `/warmup-course/` 子路径下天然可用，不需要额外处理。这一点和 `/lsm/` 一致。

workflow 里的 `sed` 只是保险：万一某个 HonKit 版本或插件输出了绝对路径 `/gitbook/...`，这一步会把它改写成 `/warmup-course/gitbook/...`。当前版本实测输出的是相对路径，`sed` 不匹配任何内容（无害）。

## 构建版本钉死

用 `npx --yes honkit@6.2.5`，不跟随 latest。HonKit 6.x 要求 Node ≥ 20.18，workflow 里用 `setup-node@v4` 指定 `node-version: 20`。**换 HonKit 大版本前先本地跑一遍**，否则哪天新版本要求 Node 22，CI 会突然红。

## 想换成 `/warmup/` 短路径

1. 改仓库名：`gh repo rename warmup -R seekerzhz/warmup-course`
2. 把 `.github/workflows/deploy.yml` 里的 `warmup-course` 全部替换成 `warmup`
3. 重新 push，触发一次构建

站点会变成 `https://docs.hanzel.top/warmup/`，旧路径失效（Pages 会给你新的 URL，`gh-pages` 分支内容不用改）。

## 本地预览

```bash
npx --yes honkit@6.2.5 serve .
# 或者只构建：
npx --yes honkit@6.2.5 build . _book && python3 -m http.server -d _book 8080
```
