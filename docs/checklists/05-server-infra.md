# Checklist 05 — Server / Infra (1 VPS ARM Singapore + Docker Compose + Caddy, sau Cloudflare)

> Dùng để AI check `infra/**`, `.github/workflows/deploy.yml`, `docs/ops/runbooks/*`; bạn re-check `[M]` (nhiều mục phải chạy từ máy ngoài). Tham chiếu D08–D10, D22; `01-CONTRACTS.md` §5, §8 Infra; số đo `03-MEASUREMENT-TOOLS.md` §5.

## 0. Biên giới: Infra nhận gì, phải trả gì

| Hướng | MUST | OPT | Nguồn |
|---|---|---|---|
| **Input** artifact | Image `ghcr.io/<org>/app:<sha>` (multi-arch, distroless) từ CI backend; tag `PREV_IMAGE` để rollback | | Checklist 03 B03–B04 |
| **Input** cấu hình | `infra/compose/.env` (từ `.env.example`, secret thật **chỉ trên VPS** trong `/opt/app/.env` chmod 600, hoặc SOPS Perfect); `Caddyfile`; `DOMAIN_API`, `ACME_EMAIL` | SOPS + age | Contracts §8 Infra |
| **Input** mạng | DNS `api.<domain>`, `api-staging.<domain>` → VPS IP (Cloudflare proxied, SSL Full strict, Origin cert) | Cloudflare IP allowlist | Checklist 06 |
| **Input** DB | `DATABASE_URL` Neon pooled (API) + direct (migrate); không Postgres trên VPS | | Checklist 04 |
| **Output** dịch vụ | `https://api.<domain>/readyz` 200; `/v1/*` proxy tới `api:8080`; `/metrics`, pprof **không** public; staging song song cùng VPS (project compose riêng, port nội bộ khác) | | |
| **Output** vận hành | Deploy 1 lệnh ≤ 3 phút có rollback tự động; alert đến điện thoại ≤ 5 phút; backup job chạy (04-S07); runbooks; metrics node + container lên Grafana Cloud | Blue/green | |
| **Output** evidence | `docs/evidence/server/*` | | |

## 1. Cấu trúc file kỳ vọng

```
infra/
├── compose/
│   ├── docker-compose.yml            # services: caddy, api, worker, alloy (Grafana agent), cadvisor?, backup (cron image)
│   ├── docker-compose.staging.yml    # override: image tag staging, DOMAIN_API=api-staging, env file khác
│   ├── Caddyfile                     # api.<domain> + api-staging.<domain>
│   └── .env.example
├── scripts/
│   ├── bootstrap-vps.sh              # idempotent: user, ssh hardening, ufw, fail2ban, unattended-upgrades, docker, dirs, swap, sysctl, timezone, cron backup
│   ├── deploy.sh                     # pull image sha → migrate (container tạm) → up api/worker → wait /readyz 60 s → rollback PREV_IMAGE nếu fail → prune
│   ├── backup.sh  restore-drill.sh   # 04-S07/S08
│   ├── cf-ip-allowlist.sh            # Perfect: ufw chỉ cho CF IP vào 443
│   └── rotate-secrets.md             # quy trình xoay từng secret
├── goss.yaml                         # spec server (S6 measurement)
├── grafana/{dashboards/{api,node,db}.json, alerts/*.yaml}
└── alloy/config.alloy                # scrape api:6060/metrics, node_exporter, cadvisor → Grafana Cloud
.github/workflows/deploy.yml          # main → build (backend.yml) → ssh deploy staging → smoke → manual approve → prod
docs/ops/runbooks/{deploy.md, rollback.md, incident.md, db-down.md, disk-full.md, cert-expired.md, ddos.md, restore.md, rotate-secrets.md, vps-rebuild.md}
docs/ops/accounts.md  docs/ops/tooling.md
```

## 2. A — VPS & bootstrap & compose (A01–A12)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| A01 | VPS: ARM (Oracle A1 4 OCPU/24 GB free hoặc Hetzner CAX11 2 vCPU/4 GB), Ubuntu LTS mới nhất, region Singapore; 1 máy chạy cả staging + prod (compose project khác nhau) | Máy staging riêng | | `docs/ops/accounts.md` ghi provider, region, spec; `uname -m` = aarch64 | |
| A02 | `bootstrap-vps.sh` **idempotent** (chạy 2 lần không lỗi): tạo user `deploy` (sudo NOPASSWD chỉ cho `docker`, `systemctl restart app`), ssh key only, `PermitRootLogin no`, `PasswordAuthentication no`, đổi port SSH (hoặc giữ 22 + fail2ban), ufw default deny in, allow `ssh, 80, 443`, fail2ban sshd, `unattended-upgrades` security auto + reboot window, Docker CE + compose plugin, `/opt/app/{compose,env,backups}`, swap 2 GB, sysctl (`net.core.somaxconn`, `fs.inotify`), timezone UTC, chrony, journald limit 500 MB, `cron`/systemd timer backup | Rootless Docker; Tailscale SSH; CIS baseline | | `goss validate` 100% (S6 evidence); chạy script lần 2 exit 0 | |
| A03 | Compose: `api` + `worker` cùng image khác command; `restart: unless-stopped`; `read_only: true` + `tmpfs: /tmp`; `cap_drop: [ALL]`; `security_opt: [no-new-privileges:true]`; `mem_limit` api 512 MB, worker 512 MB, caddy 128 MB; `healthcheck` gọi `/healthz` (dùng `wget`? distroless không có → dùng Go binary subcommand `app healthcheck` hoặc Caddy check); logging `json-file` max 50 MB × 3 | `pids_limit` | | S10 measurement `docker inspect` JSON; compose file | |
| A04 | Mạng: chỉ `caddy` publish `80:80, 443:443, 443:443/udp`; `api`, `worker`, `alloy` **không** `ports:`; network `internal` riêng; `/metrics` và pprof chỉ trong network | | | `docker compose ps` chỉ caddy có port; nmap ngoài (S3) | |
| A05 | Caddyfile: `api.<domain> { encode zstd gzip; reverse_proxy api:8080 { health_uri /readyz; health_interval 10s; lb_try_duration 5s }; header { Strict-Transport-Security "max-age=63072000; includeSubDomains"; X-Content-Type-Options nosniff; Referrer-Policy no-referrer; -Server }; handle /metrics* { respond 404 }; handle /debug/* { respond 404 }; log { output stdout; format json } }`; TLS: Cloudflare Origin cert (15 năm) **hoặc** ACME DNS-01 Cloudflare; chỉ TLS 1.2+ | Rate limit layer Caddy; `request_body max_size 1MB` | | Caddyfile; `curl -I` headers; testssl (S4) | |
| A06 | Cloudflare: proxied (cam), SSL **Full (strict)**, Always HTTPS, min TLS 1.2, HTTP/3 on, WAF managed rules free, Bot Fight Mode, rate limiting rule `/v1/auth/*` 30 req/phút/IP, cache rule bypass cho `api.` trừ `/v1/dictionary/*` (respect origin `Cache-Control`) | CF IP allowlist trên ufw (`cf-ip-allowlist.sh`), Authenticated Origin Pulls | | Screenshot từng setting; `curl -sI https://api.<domain>/v1/dictionary/search?q=a \| grep cf-cache-status` = HIT lần 2 | [M] |
| A07 | `deploy.sh` (chạy bởi CI qua SSH hoặc tay): `set -euo pipefail`; `docker pull $IMAGE`; `docker compose run --rm migrate` (dùng `DATABASE_URL_DIRECT`, fail → dừng); `docker compose up -d api worker`; poll `curl -sf http://127.0.0.1:<caddy internal>/readyz` tối đa 60 s; fail → `IMAGE=$PREV_IMAGE docker compose up -d` + exit 1; thành công → ghi `PREV_IMAGE=$IMAGE` vào `.env`; `docker image prune -f` giữ 3 tag; in timestamp mỗi bước | Blue/green 2 container + Caddy switch (0 downtime) | | S9 evidence deploy-drill: thời gian + số non-2xx; test rollback bằng image cố tình hỏng `/readyz` | [M] |
| A08 | CI `deploy.yml`: trigger sau `backend.yml` xanh trên `main` → SSH (deploy key riêng, user `deploy`, `command=` restricted trong `authorized_keys` chỉ chạy `deploy.sh`) → staging → smoke (`/readyz`, `/v1/version` sha khớp, schemathesis rút gọn) → **manual approval** (GitHub environment `production` với required reviewer = bạn) → prod cùng script | Canary % qua Cloudflare | | Workflow; `authorized_keys` có `command=`; screenshot environment protection | |
| A09 | Secrets: `/opt/app/env/{prod,staging}.env` chmod 600 owner `deploy`; không secret trong repo/CI log (`::add-mask::`); GitHub secrets chỉ: `SSH_KEY`, `SSH_HOST`, `GHCR_TOKEN`; `rotate-secrets.md` liệt kê mọi secret + chu kỳ + cách xoay không downtime (JWT 2 key song song) | SOPS + age trong repo | | `ls -l /opt/app/env`; `gitleaks` repo 0; runbook | |
| A10 | Thời gian & log: UTC; journald + docker json-file giới hạn; disk free ≥ 30%; alert disk < 20% | Loki | | `df -h`; alert rule | |
| A11 | Updates: `unattended-upgrades` security; Docker image base update qua renovate PR; Caddy image pin minor; reboot cửa sổ 04:00 UTC + alert nếu cần reboot | | | `/etc/apt/apt.conf.d/50unattended-upgrades`; renovate config | |
| A12 | Rebuild từ 0: runbook `vps-rebuild.md` — VPS mới → `bootstrap-vps.sh` → copy env từ password manager → `deploy.sh` → DNS switch; **đã diễn tập 1 lần** ≤ 30 phút | ≤ 15 phút, Terraform/cloud-init | | Runbook có timestamp drill | [M] |

## 3. S — Bảo mật server (S01–S08) — số từ `03-MEASUREMENT-TOOLS.md` §5

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| S01 | Lynis hardening index ≥ 75; 0 warning về SSH/firewall/kernel | ≥ 85 | | S1 evidence | |
| S02 | ssh-audit 0 fail; chỉ `ed25519` host key + client key; `MaxAuthTries 3`; `AllowUsers deploy`; `X11Forwarding no`; `ClientAliveInterval 300` | SSH chỉ qua Tailscale (port 22 đóng public) | | S2 evidence; `sshd -T` | |
| S03 | nmap từ ngoài: chỉ SSH port + 80 + 443 (TCP) và 443/udp (HTTP/3); không 5432/6060/9100/8080/2019 | Chỉ 80/443 | | S3 evidence | [M] |
| S04 | fail2ban jail sshd (bantime 1 h, maxretry 5) + jail caddy 4xx flood (tuỳ chọn); log có ban thật sau khi test sai pass 6 lần từ máy khác | | | `fail2ban-client status sshd` | [M] |
| S05 | Docker daemon: `"live-restore": true`, `"log-driver": "json-file"` với `max-size`; `userns-remap` hoặc rootless (Perfect); không mount `docker.sock` vào container nào (cadvisor read-only nếu dùng) | Rootless | | `/etc/docker/daemon.json`; `docker inspect` mounts | |
| S06 | Trivy `config` cho `infra/` (Dockerfile, compose, Caddyfile) 0 HIGH; dockle 0 FATAL/WARN | | | S5 evidence | |
| S07 | TLS: SSL Labs A+ cho `api.` (qua CF) và origin không public (chỉ CF hoặc Origin cert không tin bởi browser — OK); HSTS ≥ 1 năm; OCSP stapling (Caddy mặc định) | HSTS preload | | S4 evidence | |
| S08 | Kernel/sysctl: `net.ipv4.conf.all.rp_filter=1`, `net.ipv4.tcp_syncookies=1`, `kernel.kptr_restrict=2`, `fs.protected_*`; `AppArmor` enabled | | | `sysctl -a \| grep` trong goss | |

## 4. O — Observability & alerting (O01–O06)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| O01 | Grafana Alloy: scrape `api:6060/metrics` (15 s), node_exporter (host), cAdvisor (container) → Grafana Cloud Prometheus; OTLP receiver forward traces từ api → Tempo | Logs → Loki | | `config.alloy`; Grafana Explore có `up{job="api"}=1` | |
| O02 | Dashboards import từ repo `infra/grafana/dashboards/{api,node,db}.json`: RED per route, pool, River jobs; node CPU/RAM/disk/IO; Neon metrics (qua API nếu có) | | | Screenshot 3 dashboard có dữ liệu | |
| O03 | Alerts (Grafana Cloud contact point → Telegram/Email/SMS): `/readyz` down 2 phút (UptimeRobot + Grafana), 5xx > 1%, p95 > 300 ms 10 phút, disk < 20%, RAM > 90%, container restart > 3/giờ, backup ping miss (healthchecks.io), cert hết hạn < 14 ngày | On-call rotation (1 người: vẫn ghi) | | `alerts/*.yaml`; drill: `docker compose stop api` → điện thoại kêu ≤ 5 phút → ghi thời gian | [M] |
| O04 | UptimeRobot (hoặc Better Stack) monitor `https://api.<domain>/readyz` 1 phút + `https://<domain>` + `https://admin.<domain>`; healthchecks.io cho cron backup + rollup job | Status page public `status.<domain>` | | Screenshot monitors; S8 evidence uptime CSV | |
| O05 | Log: Caddy access JSON + api slog JSON ra stdout; `docker compose logs --since 1h api \| jq` parse 100%; không log body/token | Loki + alert trên log pattern | | Output jq | |
| O06 | Cost tracking: `docs/evidence/server/<date>-cost.md` hàng tháng: VPS, Neon, R2, Cloudflare, Grafana, Sentry; alert Neon storage > 80% free tier | | | File tồn tại | |

## 5. R — Runbooks & drills (R01–R06)

| ID | Mục | Target Bare | Target Perfect | Cách check | M |
|---|---|---|---|---|---|
| R01 | `deploy.md`, `rollback.md`: lệnh chính xác, điều kiện, thời gian kỳ vọng, cách xác nhận | | | File + drill S9 | |
| R02 | `incident.md`: 5 bước (xác nhận → giảm tác động → thông báo status → sửa → post-mortem template `docs/reviews/incidents/YYYY-MM-DD.md`) | | | File | |
| R03 | `db-down.md`, `disk-full.md`, `cert-expired.md`, `ddos.md` (bật Cloudflare Under Attack), `restore.md` (trỏ 04-S08) | | | 5 file tồn tại, mỗi file có "Triệu chứng / Xác nhận / Xử lý / Phòng ngừa" | |
| R04 | `rotate-secrets.md`: bảng secret × chu kỳ × cách xoay × downtime; JWT key rotation 2 key song song đã test trên staging | | | Bảng; test rotate staging: token cũ vẫn verify được 15 phút | [M] |
| R05 | Drills đã làm và có timestamp: (1) deploy + rollback, (2) stop api → alert, (3) restore DB, (4) rebuild VPS | Quarterly | | 4 mục trong `docs/evidence/server/<date>-drills.md` | [M] |
| R06 | `accounts.md`: mọi account (provider, email đăng nhập, 2FA, ai giữ recovery codes) — không secret | | | File; 2FA 100% | |

## 6. Bare Minimum vs Perfect

| Bare | Perfect |
|---|---|
| A01–A12 (1 lần drill rebuild), S01–S08 (Lynis ≥ 75), O01–O06, R01–R06 | Cloudflare IP allowlist + Authenticated Origin Pulls, Tailscale SSH (22 đóng), rootless Docker, SOPS/age secrets trong repo, blue/green zero-downtime, canary, Loki logs, status page, Terraform/cloud-init, máy staging riêng, Lynis ≥ 85 |

## 7. G — Prompt cho AI (copy nguyên văn)

```
Bạn là DevOps/SRE reviewer. Đầu vào: docs/checklists/05-server-infra.md, docs/01-CONTRACTS.md §5 và §8 (Infra),
thư mục infra/ (compose, Caddyfile, scripts, goss.yaml, grafana, alloy), .github/workflows/deploy.yml, docs/ops/**,
và evidence docs/evidence/server/ nếu có (lynis.txt, ssh-audit.json, nmap.txt, testssl.json, trivy-config.json, dockle.json,
goss.json, deploy-drill.md, drills.md, container-hardening.json, uptime.csv).
Kiểm tra từng mục A01→A12, S01→S08, O01→O06, R01→R06.
Với mỗi mục trả về đúng 1 dòng bảng:
ID | PASS/FAIL/N-A | bằng chứng (file:dòng hoặc lệnh + output rút gọn) | việc cần sửa (nếu FAIL)
Quy tắc:
- Không PASS nếu không có bằng chứng. Số (hardening index, port list, thời gian deploy) trích từ evidence.
- Mục [M] chỉ ghi "CẦN MANUAL".
- Với A03: dán JSON docker inspect (ReadonlyRootfs, CapDrop, SecurityOpt, Memory) cho api, worker, caddy.
- Với A07: đọc deploy.sh và xác nhận có: set -euo pipefail, migrate trước up, poll readyz ≤ 60 s, rollback PREV_IMAGE, ghi PREV_IMAGE mới.
- Với A02: liệt kê từng bước trong bootstrap-vps.sh và đánh dấu bước nào KHÔNG idempotent (chạy lần 2 sẽ lỗi hoặc đổi trạng thái).
- Cuối cùng liệt kê: mọi service compose có ports:, mọi container không read_only, mọi secret trong repo/CI log, mọi runbook thiếu.
- Kết thúc bằng bảng tổng PASS/FAIL/MANUAL theo nhóm.
```

## 8. Re-check thủ công (bạn làm từ laptop + điện thoại, ~90 phút lần đầu)

1. **Từ ngoài (S03)**: `nmap -sS -p- <ip>` và `nmap -sU --top-ports 50 <ip>` từ mạng khác (4G). Chỉ thấy port kỳ vọng.
2. **Alert thật (O03)**: `ssh deploy@vps 'cd /opt/app/compose && docker compose stop api'`, bấm giờ đến khi điện thoại nhận thông báo. `start` lại. Ghi số phút.
3. **Deploy + rollback (A07)**: đẩy image cố tình fail `/readyz` (env sai) → `deploy.sh` phải tự rollback, `/v1/version` sha vẫn là bản cũ. Sau đó deploy bản đúng, đo thời gian và non-2xx bằng `hey -z 3m` chạy song song.
4. **fail2ban (S04)**: từ máy khác `ssh wrong@ip` 6 lần → IP bị ban (kiểm tra `fail2ban-client status sshd`), unban sau khi test.
5. **Cloudflare (A06)**: `curl -sI https://api.<domain>/v1/version | grep -i "cf-ray\|server"`; tắt proxy (grey cloud) thử → nếu Perfect allowlist bật thì phải timeout.
6. **Rebuild (A12)**: 1 lần duy nhất trước launch, VPS tạm khác, bấm giờ.
7. **Secret (A09)**: `grep -r "eyJ\|sk_\|re_" .github infra docs` = 0; GitHub Actions log 1 lần deploy không lộ env.

## 9. Evidence phải có khi đóng Phase 1 (staging) và Phase 4 (prod)

`docs/evidence/server/`: `<date>-lynis.txt`, `-ssh-audit.json`, `-nmap.txt`, `-testssl.json`, `-trivy-config.json`, `-dockle.json`, `-goss.json`, `-node-under-load.png`, `-container-hardening.json`, `-deploy-drill.md`, `-drills.md`, `-uptime.csv`, `-cost.md`, `-cf-settings/*.png`, `checklist-05-run-<date>.md`.
