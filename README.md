# Gateway

Caddy HTTPS 入口，服务器目录 `/opt/site-gateway`；所有站点使用外部 Docker 网络 `gateway`。

| 域名 | 内部目标 |
|---|---|
| suzutsuki.com | site-main-web:3000 |
| cm.suzutsuki.com | **comike-web:3001** |
| cm-old.suzutsuki.com | comiket-web:80（旧项目保留） |
| timeline.suzutsuki.com | timeline-web:3000 |
| acc.suzutsuki.com | accounting-web:3000 |

DNS 指向服务器，开放 TCP 80/443；`.env` 配置 `ACME_EMAIL`，不提交实际环境文件。
数据库不加入 gateway 网络，网站不得另行占用宿主机 80/443。

## 更新

**先部署 Comike 并确认 `http://comike-web:3001/api/health/db` 可达，再推送此次网关修改。**
不要停止旧 `comiket-web`，`cm-old` 仍依赖它。

推送 main/master 后由 [现有 Action](.github/workflows/deploy.yml) 更新；
Secrets：`VULTR_SSH_KEY`、可选 `VULTR_SSH_PASSPHRASE`、已核实的 `VULTR_HOST_FINGERPRINT`。

手动更新：

```bash
cd /opt/site-gateway
git pull --ff-only
docker compose config --quiet
docker compose run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker compose up -d --no-deps --force-recreate caddy
curl -fsS https://cm.suzutsuki.com/api/health/db
curl -I https://cm-old.suzutsuki.com
```

不要删除证书卷或执行 `down -v`。建议 DNS-only；Cloudflare 代理须使用 Full (strict)，并配置其可信代理网段以保留真实客户端 IP。
