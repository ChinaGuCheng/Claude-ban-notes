# CPA 部署指南

> - 只适合**个人自用**：你自己的账号、你自己的设备。转发给别人、共享账号凭据，违反官方条款，可能被封号，不退款。条款见 Anthropic 的 [Consumer Terms](https://www.anthropic.com/legal/consumer-terms) 和 [Claude Code 法务页](https://code.claude.com/docs/en/legal-and-compliance)。
> - 不教你登录别人的账号。OAuth 只在你自己的浏览器里，登你自己的号。
> - 不开放注册，不做成商业服务。

CPA 即 [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)，把订阅账号的登录转成标准 API 入口。下面用 Docker 装在一台海外 Ubuntu 上，Caddy 做 HTTPS，让 Claude Code 连上来。占位符：`example.com` 域名，`<SERVER_IP>` 服务器 IP，`<CLIENT_KEY>` 客户端密钥，`<MGMT_KEY>` 管理密钥。

## 1. 准备

- 2 核 2GB 以上、20GB 磁盘的 Ubuntu 22.04/24.04。上行带宽别太小，Claude Code 每次重发整段对话，请求体可到 10MB 以上。
- 把 `example.com` 的 A 记录指向 `<SERVER_IP>`，先加记录再装 Caddy。第一次别套 CDN。
- 安全组只开 **22、80、443**，CPA 的端口只绑 `127.0.0.1`。`ufw` 拦不住 Docker 发布的端口，所以安全组和 `127.0.0.1:` 前缀两层都要有。

## 2. 装 Docker 和 Caddy

```bash
curl -fsSL https://get.docker.com -o /tmp/get-docker.sh   # 先看一眼再跑
sudo sh /tmp/get-docker.sh && sudo docker compose version
```

Caddy 按官方 apt 方式装：<https://caddyserver.com/docs/install>，装完先不改 Caddyfile。

## 3. docker-compose.yml

```bash
sudo mkdir -p /opt/cpa/{auths,logs,data,plugins,backups} && sudo chmod 700 /opt/cpa/auths /opt/cpa/backups
sudo tee /opt/cpa/docker-compose.yml >/dev/null <<'EOF'
services:
  cli-proxy-api:
    image: eceasy/cli-proxy-api:v8.0.52
    container_name: cli-proxy-api
    ports: ["127.0.0.1:8317:8317", "127.0.0.1:54545:54545"]   # 54545 是 Claude 登录回跳
    volumes: ["./config.yaml:/CLIProxyAPI/config.yaml", "./auths:/root/.cli-proxy-api", "./logs:/CLIProxyAPI/logs", "./data:/CLIProxyAPI/data", "./plugins:/CLIProxyAPI/plugins"]
    restart: unless-stopped
EOF
```

- **固定版本号，不用 `latest`。** 官方 compose 默认 `latest` 加 `pull_policy: always`，每次 `up` 都可能换版本。
- **必须挂 `data`。** 官方默认 compose 没有这一层。没挂的东西在容器可写层里，一重建容器就丢，有人的插件用量库就这样没了。

## 4. config.yaml

先生成密钥，只显示一次，存进密码管理器，别贴进聊天：
```bash
umask 077; CLIENT_KEY=$(openssl rand -hex 32); MGMT_KEY=$(openssl rand -hex 24)
printf '客户端 key: %s\n管理 key:   %s\n' "$CLIENT_KEY" "$MGMT_KEY"
sudo install -m 600 /dev/null /opt/cpa/config.yaml
sudo tee /opt/cpa/config.yaml >/dev/null <<EOF
config-version: 8
server: {host: "", port: 8317}   # host 容器内留空，写 127.0.0.1 会让端口映射失效
management: {allow-remote: true, secret-key: "${MGMT_KEY}", disable-control-panel: false}
access:
  api-keys:
    - "${CLIENT_KEY}"
routing:
  strategy: "round-robin"
  session-affinity: true
  session-affinity-ttl: "1h"
  session-affinity-rebind-on: "quota"
observability:
  logs: {debug: false, logging-to-file: true, logs-max-total-size-mb: 1024, request-log: false}
EOF
unset CLIENT_KEY MGMT_KEY
```

- `session-affinity`：同一段对话固定用一个账号。Claude 的缓存跟账号绑定，换号整段重写（见第 10 节）。
- **管理密钥泄露等于凭据泄露**：管理接口能下载登录文件。`allow-remote: true` 不等于暴露公网，靠三层：8317 只绑回环、Caddy 只转发 `/v1/*`、管理页走 SSH 隧道。
- **`config.yaml` 是单文件挂载，别用 `sed -i` 或 `mv` 改它。** 文件换 inode 后，容器里看到的还是旧文件。要改就原地写：`sudo tee /opt/cpa/config.yaml < 新文件 >/dev/null`；已经断开的，`sudo docker compose restart`。读配置别 `cat`，里面有密钥。

## 5. 启动与验证

```bash
cd /opt/cpa && sudo docker compose up -d
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8317/v1/models    # 期望 401；000 是没起来
```

CPA 没有 `/health`，健康检查用 `GET /v1/models`。带密钥测时别写在命令行里，用 `read -rs K` 读进变量，再 `curl -K -` 从标准输入传请求头。

## 6. Caddy HTTPS

只转发 `/v1/*`，其余一律 404：
```bash
sudo tee /etc/caddy/Caddyfile >/dev/null <<'EOF'
example.com {
	handle /v1/* {
		reverse_proxy 127.0.0.1:8317 {
			flush_interval -1
		}
	}
	handle {
		respond "Not Found" 404
	}
}
EOF
sudo caddy validate --config /etc/caddy/Caddyfile && sudo systemctl reload caddy
for u in v1/models management.html v8/management/config; do curl -sS -o /dev/null -w "$u %{http_code}\n" https://example.com/$u; done   # 期望 401 404 404
```

`flush_interval -1` 让流式数据立刻转发。不要把 `/v8/*`、`/management.html` 加进公网路由。证书失败看 `journalctl -u caddy`，多半是 DNS 没生效或 80/443 没放行。

## 7. 登录你自己的账号

管理页的 OAuth 按钮只支持本机运行的 CPA，远程服务器用命令行：
```bash
cd /opt/cpa && sudo docker compose exec cli-proxy-api /CLIProxyAPI/CLIProxyAPI -no-browser --claude-login
```

它会打印一条 `ssh` 隧道命令和一个授权链接：在你自己的电脑上执行那条 `ssh` 并保持窗口开着，再用电脑浏览器打开链接授权。登录文件存在 `/opt/cpa/auths`，不要 `cat`、不要提交 Git。`/v1/models` 列表非空就是成功。管理页：`ssh -N -L 8317:127.0.0.1:8317 ubuntu@<SERVER_IP>`，浏览器开 `http://127.0.0.1:8317/management.html`，输入 `<MGMT_KEY>`。

## 8. Claude Code 怎么接

地址写域名根，**不带 `/v1`**，它自己会拼 `/v1/messages`，多写会 404。
```bash
export ANTHROPIC_BASE_URL="https://example.com"
export ANTHROPIC_AUTH_TOKEN="<CLIENT_KEY>"   # 然后运行 claude
```

## 9. 升级与备份

**先备份，再停机，再换版本；停机、备份、拉起、回退写在同一条脚本里，不要单独 `docker compose stop`。** 如果你是通过这台 CPA 驱动 AI 助手来升级的，`stop` 之后你的请求就断了，后面的步骤没人执行，容器一直停着。用 `systemd-run` 启动，SSH 断开也不影响。

`/opt/cpa/upgrade.sh`，用法 `sudo systemd-run --unit=cpa-upgrade --collect bash /opt/cpa/upgrade.sh v8.0.53`：
```bash
#!/usr/bin/env bash
set -u; trap '' HUP; cd /opt/cpa || exit 1
NEW="${1:?用法: upgrade.sh <新版本标签>}"; IMG="eceasy/cli-proxy-api"
OLD=$(sed -n "s|^[[:space:]]*image:[[:space:]]*${IMG}:\([^[:space:]]*\).*|\1|p" docker-compose.yml | head -n1)
set_tag() { sed -i "s|\(image:[[:space:]]*${IMG}:\)[^[:space:]]*|\1$1|" docker-compose.yml; }
up() { docker compose up -d; for _ in $(seq 60); do   # 200 或 401 都算起来了
  [[ "$(curl -s -o /dev/null -m 3 -w '%{http_code}' http://127.0.0.1:8317/v1/models)" =~ ^(200|401)$ ]] && return 0; sleep 2; done; return 1; }
[ -n "$OLD" ] && [ "$OLD" != "$NEW" ] && docker pull "$IMG:$NEW" || { echo "没动任何东西"; exit 1; }
BK="backups/pre-$NEW-$(date +%Y%m%d-%H%M%S)"; docker compose stop
mkdir -p "$BK" && cp -a docker-compose.yml config.yaml auths data plugins "$BK"/ || { echo "备份失败"; up; exit 1; }
set_tag "$NEW"; up && { echo "成功"; exit 0; }
echo "失败，回退到 $OLD"; set_tag "$OLD"; up; exit 1
```

- 升级会清空会话绑定，每段对话重写一次缓存，挑不用的时候做。重启后头 1 秒的请求会得到 400 `unknown provider for model`，不是新版回归。
- `auths` 定期加密备份到服务器之外：`sudo tar czf - -C /opt/cpa auths | gpg --symmetric -o auths-$(date +%Y%m%d).tar.gz.gpg`。丢了只是重新登录，泄露才是事故。手工回退：compose 里标签改回去再 `up -d`；配置被新版改过就用备份原地写回，别整份 `cp` 覆盖。

## 10. 常见问题

- **401 / 403**：401 是没带密钥、头写错，或复制时带了换行空格；403 多半是上游转发的账号问题，看 `/opt/cpa/logs/main.log` 同一时刻的上游错误。
- **404**：路径变成 `/v1/v1/messages`，是 `ANTHROPIC_BASE_URL` 多写了 `/v1`；`/management.html` 404 是 Caddy 故意的，走 SSH 隧道。
- **换号后变慢、变贵**：缓存跟账号绑定，对话换到另一个账号，整段上下文重写。开 `session-affinity`，别给某个账号单独设更高优先级，别把重试次数调大。找原因：`main.log` 里搜 `failover`。
- **配了全局代理，不会回退直连。** 键是 `requests.proxy-url`。代理一断，所有请求都失败；代理时不时掐 TLS 握手（日志 `handshake upstream: EOF`），CPA 换号重试，缓存整段重写。要用代理，先对 `api.anthropic.com` 测握手，并想好一键关掉的办法。
- **上游持续 400**：若是某一批账号同一时段一起开始、另一些仍正常，可能是被封了，但**单凭 CPA 日志认不出封号**。先停用有问题的凭据，免得重试风暴，再用浏览器登录账号看状态。起点和你的升级、改配置、换出口重合，先怀疑自己的改动。

这份指南没在新机器上从零验证过，配置以官方 `config.example.yaml` 为准。
