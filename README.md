# Gateway

使用 Caddy 提供 HTTPS，并按域名转发到各网站。本项目不运行网站代码或数据库。

| 域名 | 内部目标 |
| --- | --- |
| `suzutsuki.com` | `site-main-web:3000` |
| `cm.suzutsuki.com`、`cm-old.suzutsuki.com` | `comiket-web:80` |
| `timeline.suzutsuki.com` | `timeline-web:3000` |
| `acc.suzutsuki.com` | `accounting-web:3000` |

## 准备

- 服务器安装 Docker 和 Compose，域名指向服务器，开放 TCP 80/443。
- Caddy 和网站共用外部 Docker 网络 `gateway`，网站设置上表中的网络别名。
- 只将网站服务接入此网络，保留其原有内部网络；不接入数据库。
- 仅 Caddy 对外发布端口，不同时启动另一个占用 80/443 的网关。
- HTTPS 不代表访问受限，各网站自行负责登录权限。

## 首次启动

命令在 Linux 服务器执行，现有部署目录为 `/opt/site-gateway`。

```bash
cd /opt/site-gateway
docker network inspect gateway
```

仅当网络不存在时执行 `docker network create gateway`。

```bash
test -f .env || cp .env.example .env
nano .env
```

填写 `ACME_EMAIL`，不提交真实环境文件。网站先启动，再校验和启动网关：

```bash
docker compose config --quiet
docker compose run --rm --no-deps caddy \
  caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

校验成功后：

```bash
docker compose up -d --no-deps --force-recreate caddy
docker compose logs --tail=100 caddy
curl -I https://suzutsuki.com
```

## 更新

修改 [Caddyfile](Caddyfile)，推送 main/master 后由现有 [Actions](.github/workflows/deploy.yml) 自动部署；等待成功即可，无需重复手动操作。

首次配置 Actions 时需要 `VULTR_SSH_KEY`、可选的 `VULTR_SSH_PASSPHRASE`、已核实的 `VULTR_HOST_FINGERPRINT`；服务器另需仓库读取权限。

手动更新时，在上述目录运行 `git pull --ff-only`，再执行校验，成功后重建 Caddy。重建会短暂中断入口，不重启网站。

不要删除证书卷或执行 `docker compose down -v`。Cloudflare 开启代理时使用 Full (strict)。

主站部署见 [site-main 部署指南](https://github.com/Tinellie/site-main/blob/main/docs/deployment.md)。
