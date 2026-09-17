# Free-tier cloud comparison for Evolution Go

For a WhatsApp Web gateway that must stay online 24/7 (this repo). Volume here is 50–60 messages/day. Throughput is not the constraint. Sleep, RAM, disk, and egress are.

Oracle account is already created. Use that. Do not open AWS or Azure for this workload unless you need those clouds for something else.

Limits below are as of September 2026. Providers change free tiers without much notice. Confirm on the official pages before you provision.

---

## Verdict

| Rank | Cloud | For Evolution Go |
|------|--------|------------------|
| 1 | **Oracle Cloud Always Free** | Only real always-on free VPS with enough RAM. **Use this.** |
| 2 | Google Cloud `e2-micro` | Always-on, but 1 GB RAM and 1 GB/month egress. Tight and easy to bill. |
| 3 | Azure free account | 12-month VM, then paid. Fine as a trial, not a forever host. |
| 4 | AWS free plan | Credits for 6 months (new accounts). No forever VM. Worst fit. |

Render / Railway / Fly / Vercel free are not in this list. They sleep. WhatsApp drops.

---

## Side-by-side

| | Oracle | Google Cloud | Azure | AWS |
|--|--------|--------------|-------|-----|
| Always-on free VM | **Yes, forever** | **Yes, forever** (tiny) | 12 months only | No (credits, then paid) |
| Free compute | 2 OCPU + 12 GB RAM (Ampere ARM) or 2 AMD micros | 1× `e2-micro` (~1 GB RAM) | 750 h/mo each: B1s, B2pts v2, B2ats v2 | Up to $200 credits, 6 months |
| Disk | ~200 GB block | 30 GB standard PD | 2× 64 GB SSD (12 months) | From credits |
| Egress | ~10 TB/month (Always Free networking) | **1 GB/month** from North America | 15 GB outbound (12 months) | From credits / always-free leftovers |
| Trial credit | $300 / 30 days | $300 / 90 days | $200 / 30 days | $100 + up to $100 extra |
| After trial | Always Free VMs stay | Always Free `e2-micro` stays | Convert to PAYG or VMs stop; after 12 months VM is billed | Free plan ends at 6 months or when credits run out |
| Regions | Home region only for Always Free | `us-west1`, `us-central1`, `us-east1` only | Most commercial regions | Most regions |
| Card | Yes | Yes | Yes | Yes |
| Idle reclaim | Yes (low CPU/network can get the VM reclaimed) | No typical reclaim, but billing if you leave the free shape | Billing after 12 months if left running | Billing after credits if upgraded |
| Evolution Go fit | **Good** (Postgres + app on one VM) | Poor (RAM + egress) | Trial only | Do not use as the host |

Oracle ARM quota for **Always Free tenancies** is **2 OCPU / 12 GB RAM** (cut from 4 / 24 GB in 2026). Paid/PAYG tenancies may still see a higher Ampere allowance. Stay inside 2 / 12 unless you have confirmed PAYG entitlement.

---

## What Evolution Go actually needs

| Need | Why |
|------|-----|
| Process never sleeps | `clientPointer` holds the WhatsApp socket in memory |
| Persistent disk or Postgres | Session keys live in `sqlstore` (`POSTGRES_AUTH_DB` / SQLite `dbdata`) |
| ~1 GB RAM minimum | Project docs; 512 MB PaaS is too tight with ffmpeg in the image |
| Public HTTPS | QR/pair, webhooks, passkey helper |
| Outbound WebSocket | Permanent connection to WhatsApp servers |

50–60 messages/day does not need a large CPU. It needs the VM to stay up.

---

## Oracle Cloud (recommended — you already signed up)

**Links:** [oracle.com/cloud/free](https://www.oracle.com/cloud/free/) · [Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)

### What you get

- Always Free compute for the life of the account (home region).
- Ampere A1: **2 OCPU + 12 GB RAM** total (Always Free tenancy).
- Or up to **two** AMD `VM.Standard.E2.1.Micro` instances.
- ~200 GB block volume (boot disks count against this).
- VCN, public IP, load balancer (Always Free shapes).
- Autonomous DB exists, but for this app a Postgres container on the same VM is simpler.

### Why it wins here

12 GB RAM is enough for Evolution Go + Postgres + Docker with headroom. Disk is persistent. No 15-minute sleep. No 30-day Postgres wipe. This is the only Big-cloud free tier that matches this architecture.

### Watch-outs

1. **Out of capacity** — common on Ampere. Retry another availability domain, wait, or use the AMD micro shape (weaker, still always-on).
2. **Idle reclaim** — Oracle can reclaim Always Free VMs that stay near-idle (CPU / network / memory below their threshold for days). Keep the WhatsApp process running; do not create the VM and leave it empty.
3. **ARM vs AMD** — Ampere is aarch64. Official image `evoapicloud/evolution-go` lists amd64 and arm64. If a custom build fails on ARM, use the AMD micro or build locally with `GOARCH=arm64`.
4. **Do not spend the $300 trial on extra paid shapes** unless you set a budget. Always Free is enough.
5. **Open ports** — security list / NSG: `22` (SSH), `8080` or `443` (API). WhatsApp outbound must be allowed (default egress is usually open).

### Suggested layout (one VM)

```
Oracle ARM VM (2 OCPU / 12 GB)
  ├── Docker Compose
  │     ├── evolution-go   (SERVER_PORT=8080)
  │     └── postgres:15    (evogo_auth + evogo_users)
  └── optional: Caddy/nginx for HTTPS
```

Skip MinIO, RabbitMQ, NATS. Set `DATABASE_SAVE_MESSAGES=false`, `MINIO_ENABLED=false`.

Put a budget alert at $1 so a mis-clicked paid shape cannot surprise you.

---

## Google Cloud

**Links:** [cloud.google.com/free](https://cloud.google.com/free) · [Free Cloud features](https://docs.cloud.google.com/free/docs/free-cloud-features)

### What you get

- **Always Free:** 1 non-preemptible `e2-micro` in `us-west1` / `us-central1` / `us-east1`.
- 30 GB-months standard persistent disk.
- **1 GB/month** outbound from North America (China/Australia excluded).
- Separate **$300 / 90-day** trial (bills nothing until you upgrade).

`e2-micro` is time-based: hours of all `e2-micro` VMs in those regions share one month of free hours. One VM 24/7 is the intended use.

### Why it is second, not first

- **1 GB RAM** is the documented minimum for this project. Postgres on the same box will swap or OOM.
- **1 GB egress/month** is the trap. WhatsApp traffic + webhooks + pulling a Docker image + apt updates can exceed it. Overages are billed if the account is upgraded to paid.
- US regions only. Latency from India is higher than an Oracle home region you can pick closer (Mumbai exists on OCI; GCP free VM does not).

Use GCP only if Oracle capacity never appears. Even then, run Postgres on the same tiny VM and expect pain.

---

## Azure

**Links:** [Azure free account](https://azure.microsoft.com/en-us/pricing/purchase-options/azure-account) · [Free services list](https://azure.microsoft.com/en-in/pricing/free-services)

### What you get

- **$200 credit / 30 days.**
- Then you **must switch to pay-as-you-go** to keep the 12-month free services. Card can be charged after that if you exceed free amounts.
- 12 months: **750 hours/month** each of burstable Linux/Windows VMs:
  - B1s (x86, typically 1 vCPU / 1 GB) — listing varies by page; treat as small
  - B2pts v2 (Arm)
  - B2ats v2 (AMD)
- 15 GB outbound (12 months).
- 2× 64 GB SSD managed disks (12 months).
- Some services are always-free (Functions, etc.). Those do not host this WhatsApp socket.

750 hours ≈ one VM running the whole month. A second always-on VM of the same SKU in the same bucket starts billing.

### Why not for this app

After **12 months the VM is a bill**. It is a free trial with a calendar, not Always Free compute. 1 GB class VMs are also tight for Go + Postgres.

Use Azure if you already live in Microsoft 365 and only want to learn the portal. Do not migrate Evolution Go here for “free forever.”

Set a spending cap / budget the day you convert to PAYG.

---

## Amazon Web Services

**Links:** [AWS Free Tier update (Jul 2025)](https://aws.amazon.com/blogs/aws/aws-free-tier-update-new-customers-can-get-started-and-explore-aws-with-up-to-200-in-credits/) · [Free Tier](https://aws.amazon.com/free/)

### What you get (accounts created 15 Jul 2025 onward)

- Choose **free account plan** at signup.
- **$100 credits** at signup, up to **$100 more** for completing onboarding/service use (**$200 cap**).
- Plan lasts **6 months or until credits hit zero**, whichever first.
- Some always-free service slices (Lambda, DynamoDB, etc.). **Not a 24/7 EC2 that stays free forever.**
- Unused credits can last up to 12 months from signup **after you upgrade to a paid plan**.

### Legacy accounts (before 15 Jul 2025)

12-month 750 hours/month of `t2.micro` / `t3.micro`, then billed. Still not forever.

### Why it is last here

A new AWS free plan is a **time-boxed credit wallet**. An always-on `t3.small`/`t4g.small` will burn credits, then the free plan ends. EC2 left running on a paid plan is how people get surprise bills.

Do not host Evolution Go on AWS free tier. Use AWS later if you need SES, S3, or other paid services — not as the WhatsApp VM.

---

## Which is better — short

| Question | Answer |
|----------|--------|
| Best free forever VM? | **Oracle** |
| Best among “Big 3” only? | **Google** (`e2-micro`), with RAM/egress risk |
| Best 12-month classroom? | **Azure** |
| Best catalog / later paid production? | **AWS** (pay for it; do not use free as the host) |
| Best for this WhatsApp API? | **Oracle**, which you already have |

Google is the only Big-3 always-free VM. It is still a worse machine than Oracle Always Free by a wide margin (1 GB vs 12 GB).

---

## Do not do this

- Render / Railway / Fly free: sleep after idle.
- Putting the WhatsApp session on free Render Postgres (30-day expiry).
- Running two always-on Azure B-series VMs (second one bills).
- Using GCP `e2-micro` plus Cloud SQL (Cloud SQL is paid).
- Leaving AWS EC2 running after credits with a paid plan and no budget alert.
- Spending Oracle $300 trial on GPU / extra shapes “because it is free this month.”

---

## If you stay on Oracle (do this next)

1. In the Oracle console, confirm you are on **Always Free** and pick a home region with capacity (Mumbai `ap-mumbai-1` if offered).
2. Create a VCN + public subnet + NSG: SSH 22, HTTP 80, HTTPS 443, app 8080.
3. Launch **Ampere A1 Flex**: 2 OCPU, 12 GB, Ubuntu 22.04/24.04 ARM. If capacity fails, try another AD or AMD micro.
4. Attach the boot volume (counts toward 200 GB). Enable **auto-restart**.
5. SSH in, install Docker + Compose, copy `docker/examples/docker-compose.yml`, set `.env` from `.env.example`.
6. `CONNECT_ON_STARTUP=true`, `DATABASE_SAVE_MESSAGES=false`, `MINIO_ENABLED=false`.
7. Point a cheap domain + Caddy or Cloudflare Tunnel at port 8080.
8. Open `/manager`, activate the Evolution license, create one instance, scan QR.
9. Create an OCI **budget = $1** and email alert.

That is the production path for 50–60 messages/day at $0 compute, with the usual Oracle caveats (capacity, idle reclaim, unofficial WhatsApp Web).

---

## Official docs

- Oracle Always Free: https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm
- Google Free Cloud features: https://docs.cloud.google.com/free/docs/free-cloud-features
- Azure free services: https://azure.microsoft.com/en-in/pricing/free-services
- AWS Free Tier (2025 credits): https://aws.amazon.com/blogs/aws/aws-free-tier-update-new-customers-can-get-started-and-explore-aws-with-up-to-200-in-credits/
