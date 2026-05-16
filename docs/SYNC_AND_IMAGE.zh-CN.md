# 9Router 汉化镜像同步说明

本仓库用于在保留中文文档与自定义 CI 的前提下，自动跟随上游 `decolua/9router` 更新，并持续构建最新 Docker 镜像。

## 同步机制

- 工作流：`.github/workflows/sync-upstream.yml`
- 触发方式：
  - 每小时第 17 分钟自动执行一次
  - 支持 GitHub Actions 手动触发
- 同步逻辑：
  - 从 `https://github.com/decolua/9router.git` 拉取 `master`
  - 将上游提交合并到当前仓库分支
  - 若有新提交，自动推送到当前仓库

说明：

- 该方案不会做强制覆盖，因此会尽量保留本仓库自己的文档与 workflow 定制。
- 如果上游以后修改了同一文件且产生冲突，GitHub Actions 会失败，需要人工处理冲突后再继续。

## 镜像构建机制

- 工作流：`.github/workflows/docker-publish.yml`
- 构建触发：
  - `master` / `main` 分支 push
  - `v*` tag push
  - 手动触发
- 镜像仓库：
  - `ghcr.io/<GitHub用户名>/<仓库名>`

默认标签策略：

- `latest`：默认分支最新版本
- `<branch>`：分支名标签，例如 `master`
- `sha-<commit>`：提交哈希标签
- `vX.Y.Z`：语义化版本 tag

## 首次配置

1. 在 GitHub 上启用 Actions。
2. 确认仓库 Packages 权限可用，便于推送到 GHCR。
3. 如需让镜像公开可拉取，在 GitHub 仓库的 Packages 页面将容器包设置为 public。

## 使用方式

拉取最新镜像：

```bash
docker pull ghcr.io/<GitHub用户名>/<仓库名>:latest
```

运行容器：

```bash
docker run -d \
  --name 9router \
  -p 20128:20128 \
  -v 9router-data:/app/data \
  ghcr.io/<GitHub用户名>/<仓库名>:latest
```
