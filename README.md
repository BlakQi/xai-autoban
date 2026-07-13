# xai-autoban

`xai-autoban` 是一个 [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) 原生插件，用于自动隔离持续返回错误的 xAI OAuth 凭据，避免 CPA 在大号池中逐个重试无效账号而显著拉长首 token 时间。

## 功能

| 上游状态 | 默认处理 |
| --- | --- |
| `401` | 隔离 24 小时 |
| `402` | 隔离 7 天 |
| `403` | 隔离 24 小时 |
| `429` | 优先按 `Retry-After` 或限流重置头解禁，缺失时隔离 30 分钟 |

- 只处理 `xai` provider，不影响 Codex、Claude、Gemini 等其他凭据。
- 在 CPA 调度阶段跳过被隔离的凭据。
- 提供实时统计、搜索、筛选、分页和批量解禁界面。
- 支持单个、选中项、按状态码和全部解禁。
- 保留带 CPA 管理密钥保护的 Management API。

## 构建

需要 Go 1.21+、CGO 和 C 编译器。

```bash
bash build.sh
```

为确保与 CLIProxyAPI 官方 Debian 镜像兼容，推荐使用 Debian 12 构建：

```bash
docker run --rm \
  -v "$PWD:/src" \
  -w /src \
  golang:1.24-bookworm \
  sh -c 'go test ./... && CGO_ENABLED=1 go build -buildmode=c-shared -trimpath -ldflags="-s -w" -o dist/xai-autoban-linux-amd64.so .'
```

## 安装

把动态库放入 CPA 插件目录：

```text
plugins/linux/amd64/xai-autoban.so
```

在 `config.yaml` 中启用：

```yaml
plugins:
  enabled: true
  configs:
    xai-autoban:
      enabled: true
      priority: 200
```

重启 CLIProxyAPI 后，日志应包含：

```text
pluginhost: plugin registered plugin_id=xai-autoban plugin_name=xai-autoban version=0.2.0
```

## 管理面板

```text
http://<CPA_HOST>:8317/v0/resource/plugins/xai-autoban/status
```

面板不需要 CPA 管理密钥，可以查看凭据标识并执行解禁。请只在受信网络中开放该路径，或在反向代理层增加访问控制。

公开 Resource API：

```text
GET /v0/resource/plugins/xai-autoban/data
GET /v0/resource/plugins/xai-autoban/action?op=unban&auth_id=<AUTH_ID>
GET /v0/resource/plugins/xai-autoban/action?op=unban-status&status=403
GET /v0/resource/plugins/xai-autoban/action?op=unban-many&auth_ids=<ID1>,<ID2>
GET /v0/resource/plugins/xai-autoban/action?op=unban-all
```

带 CPA 管理鉴权的兼容 API：

```text
GET  /v0/management/plugins/xai-autoban/bans
POST /v0/management/plugins/xai-autoban/unban
POST /v0/management/plugins/xai-autoban/unban-all
POST /v0/management/plugins/xai-autoban/import
```

## 状态说明

- 隔离状态保存在插件进程内存中，CPA 重启后会清空。
- `402` 凭据充值或恢复订阅后，可从面板手动解禁，无需等待 7 天。
- 插件仅处理已经被请求命中的错误凭据，不会主动扫描整个账号池。
- 如果所有候选凭据都被插件隔离，最终行为交由 CPA 自带调度器决定。

## 致谢

本项目参考并改编自 [ysxk/codex-429-autoban](https://github.com/ysxk/codex-429-autoban)，并沿用了其本地化 CPA 插件 ABI 的实现思路。原项目与本项目均采用 MIT License。

## License

[MIT](LICENSE)
