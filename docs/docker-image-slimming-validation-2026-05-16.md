# Docker 镜像减重验证 2026-05-16

## 修改目标

- 在不改变运行行为的前提下缩小 Docker 镜像体积
- 保持现有 Next.js standalone 产物和 MITM 运行链路不变

## 采取的修改

- `Dockerfile` 构建阶段改为 `npm install --omit=optional`
  - 不安装 `better-sqlite3` 这类可选原生依赖
  - 当前项目已有 `sql.js` 作为运行时回退
- 去掉 `apk upgrade`
  - 避免构建和运行阶段额外拉入升级后的系统包层
- 保留 `node-forge`、`next`、`open-sse`、`src/mitm`
  - 这些仍属于当前运行链路需要的文件

## 验证命令

```powershell
Get-Content Dockerfile -Raw
docker build -t 9router-zh-slim-test .
```

## 结果摘要

- Dockerfile 已完成低风险减重修改
- 主要减重来源是去掉可选原生依赖与多余系统升级层
- 本次未继续激进删除运行文件，避免把镜像减小建立在运行时回归风险之上

## 风险说明

- 本地尚未实际执行容器启动冒烟验证时，不能把“更小”视为“完全无风险”
- 如果后续确认 `next` 已稳定包含于 standalone 追踪产物，可进一步评估是否去掉手动复制的 `node_modules/next`
