# 出站 HTTP 代理（v0.12.0）

> 让 Docker 部署的 FreeBuff2API 通过环境变量走 HTTP/HTTPS 代理转发到上游 codebuff + OpenRouter。

## 背景

`internal/app/proxy.go` 的 `NewProxyHandler` 自己构造了一个 `http.Transport`（带 TLS、Keep-Alive、HTTP/2 等显式调优），但**没有**设置 `Proxy` 字段。

Go 标准库的语义是：**自定义 Transport 的 `Proxy` 字段为 nil 时,不会读 `HTTPS_PROXY`/`HTTP_PROXY`**。只有 `http.DefaultTransport` 或显式指定 `Proxy: http.ProxyFromEnvironment` 才会读环境变量。

结果：Docker 容器里设了 `HTTPS_PROXY=...` 也完全没用,所有上游请求依然直连。

## 实现

`proxy.go:58` 加一行 `Proxy: http.ProxyFromEnvironment`,标准库立刻接管:

```go
Transport: &http.Transport{
    Proxy: http.ProxyFromEnvironment,  // ← 新增
    DialContext: ...,
    ...
}
```

`forwardToOpenRouter`（openrouter.go）和 `sessionManager`（session.go）都复用同一个 `httpClient` 实例,所以**代理设置会同时作用于**：

| 出站路径 | 涉及域名 | 是否走代理 |
|---|---|---|
| codebuff `/api/v1/chat/completions` | `www.codebuff.com` | ✅ |
| codebuff `/api/v1/agent-runs` (startAgentRun) | `www.codebuff.com` | ✅ |
| codebuff session/instance 创建 | `www.codebuff.com` | ✅ |
| OpenRouter 兜底/直转 | `openrouter.ai` | ✅ |

## 配置（环境变量,Docker 友好）

```yaml
# docker-compose.yml
environment:
  - HTTPS_PROXY=http://host.docker.internal:7890
  - HTTP_PROXY=http://host.docker.internal:7890
  - NO_PROXY=localhost,127.0.0.1,172.17.0.1
extra_hosts:
  - "host.docker.internal:host-gateway"   # Linux 必需,Mac/Win 自动有
```

支持的 scheme：**`http://`、`https://`**。**不支持 `socks5`**——故意省略以保持依赖最小化（标准库直接支持 socks5 但需要额外校验设计,本次未做）。

环境变量优先级（Go `httpproxy` 包决定）：
- 大写优先：`HTTPS_PROXY` > `https_proxy`
- 协议匹配：HTTPS 请求查 `HTTPS_PROXY`,HTTP 请求查 `HTTP_PROXY`
- `NO_PROXY` 列表里的主机直连,支持后缀匹配（`.example.com`）和 CIDR

## 已知限制

### ❌ 不支持热加载

`http.ProxyFromEnvironment` 内部用 `sync.Once` 缓存环境变量解析结果:

```go
// Go stdlib net/http/transport.go
var envProxyOnce sync.Once
var envProxyFuncValue func(*url.URL) (*url.URL, error)
```

首次有出站请求时解析,之后**改环境变量也不会重新读**。

→ 改完代理配置必须 `docker compose restart freebuff2api`,无法像 config.yaml 那样秒级热加载。

### ❌ 不支持 socks5

虽然 Go 标准库 `http.Transport.Proxy` 原生支持 `socks5://` 和 `socks5h://` scheme,但本次实现不开放。如有需求,加一个 `upstream.proxy_url` config 字段 + 自定义 Proxy 函数即可（参考 issue 历史）。

### ❌ 不支持 config.yaml 配置

代理是部署层面的运维配置,跟"哪些上游账号能用"不同。放在 docker-compose 的 environment 里比放 config.yaml 更符合 12-factor 原则,也方便不同环境（开发/staging/prod）用不同代理。

## 测试

`internal/app/proxy_test.go:TestTransportHonoursProxyEnv` 用反射比对 `Transport.Proxy` 函数指针是否等于 `http.ProxyFromEnvironment`。这是契约测试——任何"换成自定义 proxy 函数"的修改都会破这个测试,提醒开发者重写测试逻辑。

## 文件清单

| 文件 | 改动 |
|---|---|
| `internal/app/proxy.go` | 加 `Proxy: http.ProxyFromEnvironment` |
| `internal/app/proxy_test.go` | 新增 `TestTransportHonoursProxyEnv` |
| `docker-compose.yml` | 加注释化的 `environment` + `extra_hosts` 示例 |
| `docs/memory/http-proxy.md` | 本文档 |
| `docs/MEMORY.md` | changelog 一行 |
