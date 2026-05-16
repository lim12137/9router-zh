# 中文说明：上游同步与汉化镜像发布

本文档说明这个中文同步仓库如何自动跟随上游 `decolua/9router` 更新，并构建最新 Docker 镜像。

## 当前方案

- 上游代码源：`https://github.com/decolua/9router`
- 同步仓库：`https://github.com/lim12137/9router-zh`
- 镜像发布目标：`ghcr.io/lim12137/9router-zh`

## 自动同步上游

工作流文件：`.github/workflows/sync-upstream.yml`

触发方式：

- 每小时第 17 分钟自动执行
- 支持手动执行 `workflow_dispatch`

执行逻辑：

1. 拉取当前仓库完整历史
2. 添加 `decolua/9router` 为 `upstream`
3. 拉取 `upstream/master`
4. 合并到当前分支
5. 自动推送回当前仓库

说明：

- 该流程只会在定时任务和手动触发时运行，不会因为自身 push 再次触发同步死循环。
- 如果上游与本仓库对同一位置发生真实冲突，Action 会失败，此时需要人工处理冲突。

## 自动构建 Docker 镜像

工作流文件：`.github/workflows/docker-publish.yml`

触发方式：

- `master` / `main` 分支 push
- `v*` tag push
- 手动执行 `workflow_dispatch`

发布行为：

- 登录 GitHub Container Registry
- 构建多架构镜像：`linux/amd64`、`linux/arm64`
- 推送到 `ghcr.io/${{ github.repository }}`

默认标签：

- `latest`
- 分支标签，例如 `master`
- `sha-<commit>`
- 语义化版本标签，例如 `v0.4.50`

## 本地手动验证

```bash
docker build -t 9router-zh .
docker run --rm -p 20128:20128 9router-zh
```

## 相关文件

- `README.zh-CN.md`
- `.github/workflows/sync-upstream.yml`
- `.github/workflows/docker-publish.yml`
- `docs/SYNC_AND_IMAGE.zh-CN.md`
