# Kora API — container, CI/CD, and AWS (DeployReady)

## Context

Kora’s Node API in [app/](app/) runs in Docker, gets built and tested on every push to `main` in GitHub Actions, and is deployed to an **EC2** instance on AWS. The course brief also allows other clouds; I used **EC2** and **GitHub Container Registry (GHCR)** because that matched the path I was following and worked well with SSH-based deploys.

[DEPLOYMENT.md](DEPLOYMENT.md) has the full setup (security group, secrets, checklists) with **inline screenshots** next to the section they match—GitHub Actions with the stack overview, the EC2 security group, SSH + Docker on the instance, rollback, and a failed deploy example.

---

## How the pipeline works

1. Push to `main`.
2. Actions runs `npm test`, then builds the image, tags it with the **commit SHA**, and pushes to GHCR.
3. The same workflow SSHs to the instance (using repo secrets for host, user, and key) and runs `docker pull` / `docker run` for a container named `kora`, **port 80** on the host to **3000** in the app.
4. After deploy, the script checks `http://127.0.0.1/health` on the box. If that fails, it tries to start the **previous** image again and the job still fails in Actions so the bad release is obvious. Details and how I tested that are in [DEPLOYMENT.md](DEPLOYMENT.md#9-bonus-rollback-in-the-pipeline).

**Why I set it up this way**

- Commit SHA on the image tag ties what runs in prod to a specific Git commit.
- Tests run before the image is built for deploy in the same workflow.
- GHCR is enough for the registry; I didn’t need ECR in GitHub for the push.
- The rollback part was the optional bonus: catch a bad image after it lands, not only in tests.

---

## Repo layout

| Path | What it is |
|------|------------|
| [app/](app/) | API code and Jest tests |
| [Dockerfile](Dockerfile) | Image; non-root user; `PORT` |
| [docker-compose.yml](docker-compose.yml) + [.env.example](.env.example) | Local run on 3000 |
| [.github/workflows/deploy.yml](../../.github/workflows/deploy.yml) | Pipeline (repo root; `DEPLOY_ROOT` is this folder) |
| [DEPLOYMENT.md](DEPLOYMENT.md) | AWS, SG, secrets, and troubleshooting |

---

## Local run

```bash
cd app && npm install && npm test
```

With Docker (from this directory):

```bash
cp .env.example .env
docker compose up --build
```

`http://localhost:3000/health` should return `{"status":"ok"}`.

**Metrics** — `GET /metrics` returns JSON (uptime in seconds, memory MB, Node version). To try it: open `http://localhost:3000/metrics` or `curl` that URL. On the deployed server, same path on port 80: `http://<public-ip>/metrics` (see [DEPLOYMENT.md](DEPLOYMENT.md#8-checking-it)).

---

## Checklist

- [ ] `docker compose up --build` works; `.env` is gitignored
- [ ] Green run on `main` in GitHub Actions (test → build/push → deploy)
- [ ] `GET` on my public `http://<ip>/health` returns 200 and `"ok"`
- [ ] SSH is not `0.0.0.0/0` on 22; HTTP 80 can be from anywhere
- [ ] No keys or tokens in the repo (only in GitHub Secrets)
- [ ] [DEPLOYMENT.md](DEPLOYMENT.md) matches my real setup

**Submission** — fork URL through the [AmaliTech form](https://forms.cloud.microsoft.com/e/f3FF83LVz3) as required.

---

## Terms (if useful)

- **CIDR** — e.g. `203.0.113.10/32` is “just this one IP” for a firewall rule.
- **GHCR** — `ghcr.io/<user or org>/<image>:<tag>`.
- **Security group** — AWS firewall rules on the instance.

For anything that broke in practice, see [DEPLOYMENT.md](DEPLOYMENT.md).
