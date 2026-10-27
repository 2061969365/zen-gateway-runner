# zen-gateway-runner

把 `zen-direct` 网关跑在 GitHub Actions 的 runner 上，用 Cloudflare Tunnel 映射成公网 HTTPS 地址。

核心动机：上游免费额度按**出口 IP** 记账。一台 runner 就是一个 IP，IP 被 429 了就换一台
（新 run = 新 runner = 新 IP），4 小时无论如何换一次，链路靠 workflow 自派发 + cron 兜底不断。

```
客户端 -> https://<你的域名>/v1  ->  Cloudflare 隧道 -> runner:8081 (zen-direct) -> opencode.ai
```

## 一次性配置

### 1. Cloudflare

1. Zero Trust → Networks → Tunnels → **Create a tunnel**（Cloudflared 类型）。
2. 加一条 Public Hostname：
   - Subdomain / Domain：随便，例如 `zen.example.com`
   - Service：`HTTP` → `localhost:8081`
3. 复制 tunnel token（`eyJ...` 那串，`cloudflared service install` 用到的整段）。

### 2. 本仓库

- Secret：`CF_TUNNEL_TOKEN` = 上面复制的 token
- Variable：`ZEN_PUBLIC_HOST` = 你的域名，例如 `zen.example.com`（不带 `https://`）

`Settings → Secrets and variables → Actions`。少了 secret 会退化成临时隧道（`trycloudflare.com`，
每次跑完就换地址，只适合试跑）。

### 3. 客户端

```
baseURL = https://zen.example.com/v1
apiKey  = public        # 网关不校验，随便填
model   = deepseek-v4.1-flash-free
```

## 生命周期

一次 run 做的事：

1. 直接用仓库里的预编译二进制 `bin/zen-direct`（Linux amd64，9.5 MB，
   从 [`zen-direct-v2`](https://github.com/2061969365/zen-direct-v2) `24c2677` 编出来）。
   **runner 上不编译、不装 Go**，只有 `chmod +x` 和一次可执行性自检。
2. `cloudflared` 起命名隧道，等公网 `/health` 返回 200（证明映射真的通了）。
3. 从公网地址打一条**真 chat**（`deepseek-v4.1-flash-free`，`max_tokens=8`）必须拿 200，
   一次同时验证隧道和这个 IP 的额度；没拿到就判定不可用。
4. 之后每 5 分钟探一次，同时盯网关日志里的 429。
5. 满足任一条件就交班：起手探测 429 / 日志命中 429 / 连续 3 次探测非 200 / 网关反复崩溃 /
   跑满 3h50m / 隧道反复断开。
6. 先 `SIGTERM` 收掉 cloudflared（等它真的退出，最多 15s 再 `SIGKILL`），再停网关——
   保证下一个 run 接同一个命名隧道时，Cloudflare 那边已经没有这个旧 connector 了。
7. 收尾步骤 `workflow_dispatch` 下一次运行；`timeout-minutes: 240` 是硬兜底，
   每 20 分钟的 cron 再兜一层（并发组保证不会同时跑两份）。

### 429 的判定口径

**不是「日志里出现 429 就换」**，只认这两个签名：

| 来源 | 命中条件 |
| --- | --- |
| 探测响应 | HTTP `429`，或响应体含 `FreeUsageLimitError` |
| 网关日志 | `FreeUsageLimitError`，或 `free-usage 429 … turns=N` |

上游返回普通 `too many requests` / `rate limit`（非免费额度）时，网关按原样透传、不记毒，
脚本也不会因此换机。命中免费额度 429 才判死：不重试、不冷却、不等防抖，直接换 IP。

### 交班有多快

- 不编译：二进制直接在仓库里，checkout 完就能跑。
- 429 交班没有等待，清理完就走；交班原因不明的失败才守 180s 防抖。
- 新 run 的固定开销 ≈ runner 分配（10–20s）+ checkout（2–3s）+ 隧道（~8s）+ 起手探测（~2s），
  大约 **25–40s** 恢复服务；其中 runner 分配是 GitHub 的硬开销，压不下去。

## 更新二进制

改源码后重新编一份，替换 `bin/zen-direct` 再 push 即可（下一次 run 生效）：

```bash
git clone https://github.com/2061969365/zen-direct-v2
cd zen-direct-v2
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 \
  go build -buildvcs=false -trimpath -ldflags "-s -w" -o ../zen-gateway-runner/bin/zen-direct .
```

在 Windows 上 commit 时要确认可执行位没丢：

```bash
git update-index --add --chmod=+x bin/zen-direct
```

## 手动操作

- 启动：Actions → `zen-gateway` → Run workflow
- 停掉：Cancel 当前 run。**取消不会触发交班**，但 20 分钟后的 cron 会把它拉起来；
  要彻底停就把 workflow 停用（Actions → 左侧 `zen-gateway` → `…` → Disable workflow）。
- 快速验证：Actions → `smoke` → Run workflow。它**只起临时隧道**（不碰你的命名隧道），
  走公网探活，约 3 分钟退出，不碰主链路的并发组。

## 可调项

都在 `gateway.yml` 的 `env:` 里：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `PORT` | `8081` | 网关监听端口，要和 Cloudflare Public Hostname 对上 |
| `PROBE_MODEL` | `deepseek-v4.1-flash-free` | 探活用的模型 |
| `PROBE_INTERVAL_SEC` | `300` | 周期探活间隔 |
| `MAX_RUN_SEC` | `13800` | 主动交班时间（3h50m），硬超时 4h |

`ZEN_PUBLIC_HOST` 是仓库 Variable，可以留空：留空时会尝试从 cloudflared 拉到远端配置里
的 `hostname` 自动补，补不到就跳过公网自检。

## 已知边界

- 仓库是 public，标准 runner 不计费；换成 private 会吃分钟数（4h × 每天 6 次很快就穿）。
- **没有鉴权**：拿到地址的人都能用你的额度。要收紧就在 Cloudflare 侧加 Access 或 WAF 规则。
- 额度按 IP 记账，runner 的机房 IP 未必一直可用；连续 429 时会一直换，属正常消耗。
- GitHub 的 Actions 条款不允许把它当长期通用算力/代理，账号有被限制的风险，自己权衡。
- 公开仓库的定时 workflow 在仓库 60 天无活动后可能被停用；不过主链路是自派发，cron 只是兜底。
