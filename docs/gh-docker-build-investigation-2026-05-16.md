# GitHub Actions 镜像构建耗时排查与优化 2026-05-16

## 背景

- 目标：用 `gh` 直接查看 GitHub Actions 日志，确认镜像构建为什么慢，并给出可落地优化。
- 工作流：`.github/workflows/docker-publish.yml`
- 观察对象：最近两次成功的 `Build and Push Docker Image`

## 调查命令

```powershell
gh run list --limit 10
gh run view 25955755341 --json jobs,name,number,displayTitle,headBranch,createdAt,updatedAt,conclusion,url
gh run view 25954096215 --json jobs,name,number,displayTitle,headBranch,createdAt,updatedAt,conclusion,url
gh run view 25955755341 --job 76302015905 --log > $env:TEMP\gh-build-25955755341.log
gh run view 25954096215 --job 76297434718 --log > $env:TEMP\gh-build-25954096215.log
rg -n "#\d+ \[|npm install|next build|exporting|pushing|importing cache manifest|exporting cache|linux/arm64|linux/amd64" $env:TEMP\gh-build-25955755341.log
rg -n "#\d+ \[|npm install|next build|exporting|pushing|importing cache manifest|exporting cache|linux/arm64|linux/amd64" $env:TEMP\gh-build-25954096215.log
git diff -- Dockerfile .github/workflows/docker-publish.yml
```

## 结果摘要

- 最近两次镜像工作流总耗时分别为 `15m03s` 和 `16m26s`。
- 两次运行里，除 `Build and push` 外的步骤总共只占十几秒，瓶颈完全在 Docker Buildx。
- 单机同时构建 `linux/amd64,linux/arm64`，其中 `arm64` 明显最慢。

### 关键耗时证据

`25955755341`：

- `linux/amd64 npm install`：`34.0s`
- `linux/amd64 next build --webpack`：`83.1s`
- `linux/arm64 npm install`：`153.1s`
- `linux/arm64 next build --webpack`：`650.2s`
- `exporting cache to registry`：`47.3s`

`25954096215`：

- `linux/amd64 npm install`：`38.2s`
- `linux/amd64 next build --webpack`：`87.7s`
- `linux/arm64 npm install`：`165.7s`
- `linux/arm64 next build --webpack`：`718.1s`
- `exporting cache to registry`：`42.3s`

## 根因判断

1. 真正拖慢的是 `arm64` 构建，不是 checkout、登录或推镜像。
2. 现在的 workflow 在单个 `ubuntu-latest` x64 runner 上直接做双架构 build，`arm64` 构建阶段本质上成了最慢路径。
3. 最重的一段是 `arm64` 上的 `next build --webpack`，连续两次都在 `10 到 12 分钟`。
4. 远端运行日志显示，实际仍在使用旧 Dockerfile：
   - 还在跑 `apk --no-cache upgrade`
   - 还在跑全量 `npm install`
5. 本地 Dockerfile 里已经有更轻的改法，但还没进入远端构建结果。

## 已实施优化

### 工作流

- 把单 job 双架构改成矩阵并行：
  - `linux/amd64` 继续跑在 `ubuntu-latest`
  - `linux/arm64` 改跑在原生 `ubuntu-24.04-arm`
- 每个平台先按 digest 推到 GHCR。
- 最后由单独的 `publish-manifests` job 汇总 digest，生成 GHCR 和 Docker Hub 的多架构 manifest。
- cache 改为按平台分开：
  - `buildcache-amd64`
  - `buildcache-arm64`
- `actions/checkout` 不再强制检出 default branch，改为跟随当前事件 SHA，避免 tag 构建与实际发布提交不一致。

### Dockerfile

- 去掉 `apk upgrade`
- 安装依赖改为 `npm install --omit=optional`
- runner 阶段安装 `su-exec` 时也去掉 `apk upgrade`

## 预期收益

- 最大收益来自把 `arm64` 从 x64 单机双架构串行链路里拆出来，避免最慢平台卡住整条流水线。
- 结合原生 `arm64` runner，`arm64 next build` 理论上应远低于当前 `650s 到 718s`。
- Dockerfile 轻量化主要是补掉几十秒级的浪费，不是主因，但应该一并保留。

## 风险与说明

- 这次没有直接触发新的 tag workflow，所以还没有新的线上耗时数据。
- 优化基于现有两次成功运行日志，证据充分，但最终收益仍需下一次 tag 或手动 dispatch 验证。
- 如果仓库没有可用的 `ubuntu-24.04-arm` 配额或权限，需要退回到 Docker 官方的 distributed builder reusable workflow。
