# Docker 镜像减重验证 2026-05-16

## 修改目标

- 在不改变运行行为的前提下缩小 Docker 镜像体积
- 保持现有 Next.js standalone 产物和 MITM 运行链路不变

## 当前结论

- `apk upgrade` 可以安全去掉
  - 避免构建和运行阶段额外拉入升级后的系统包层
- `npm install --omit=optional` 不能用于当前 builder 阶段
  - 会导致构建缺少 `lightningcss` 的 musl 原生模块
  - 也会导致 `better-sqlite3` 在 Next 构建期被静态解析时报缺失
- 当前 builder 采用完整 `npm install`
  - 先保证 GitHub Actions 构建恢复
  - 保留构建所需原生依赖
  - 仓库当前未跟踪根 `package-lock.json`，所以暂不切到 `npm ci`
- 运行镜像仍然通过 standalone 复制保持精简
  - 最终镜像并不会把 builder 的完整 `node_modules` 原样带进去

## 验证命令

```powershell
Get-Content Dockerfile -Raw
docker build -t 9router-zh-slim-test .
```

## 结果摘要

- Dockerfile 现在采用更稳的 builder 安装策略
- 已确认 `--omit=optional` 会直接打坏 GitHub Actions 镜像构建
- 当前保留的有效优化是去掉多余 `apk upgrade`
- 镜像精简应继续建立在 standalone 产物边界上，而不是删掉 builder 必需依赖

## 风险说明

- 本地尚未实际执行容器启动冒烟验证时，不能把“能构建”视为“完全无风险”
- 如果后续确认 `next` 已稳定包含于 standalone 追踪产物，可进一步评估是否去掉手动复制的 `node_modules/next`
