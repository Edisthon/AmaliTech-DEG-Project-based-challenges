# Kora API — container, CI/CD, and AWS (DeployReady)

**Kora Analytics** (challenge scenario) needs the Node API in [app/](app/) delivered as a **container**, built and tested in **GitHub Actions**, and run on **AWS EC2** behind a public **HTTP** endpoint.

This README is the submission **architecture and operator guide** for this folder. Step-by-step AWS and troubleshooting notes are in [DEPLOYMENT.md](DEPLOYMENT.md).

---

## Architecture

1. **Developers** push to `main`.
2. **GitHub Actions** runs `npm test`, builds a **Docker** image, tags it with the **commit SHA**, and pushes to **GitHub Container Registry (GHCR)**.
3. The workflow **SSHs to EC2** (secrets: host, user, private key) and runs `docker pull` and `docker run` for container **`kora`** on **host port 80** → app port **3000**.
4. A **post-deploy** request to `http://127.0.0.1/health` on the server validates the release. If it fails, the script **rolls back** to the image that was running before (see [DEPLOYMENT.md — §9](DEPLOYMENT.md#9-bonus-rollback-in-the-pipeline); **how we tested** this is in [§9.1](DEPLOYMENT.md#91-how-we-tested-rollback-what-we-did)).

**Why this shape**

- **SHA tags** make production images **immutable** and traceable to Git.
- **Tests before deploy** stop broken code from being built for production (at least in the same pipeline).
- **GHCR** avoids storing AWS ECR keys in the repo; only GitHub and optional PAT for private pulls.
- **Rollback** is a practical bonus: bad containers do not stay live without a clear CI failure.

---

## Repository map (this challenge)

| Path | Purpose |
|------|---------|
| [app/](app/) | Node.js API (unchanged app logic; tests in-repo). |
| [Dockerfile](Dockerfile) | Production image; non-root user; `PORT` env. |
| [docker-compose.yml](docker-compose.yml) + [.env.example](.env.example) | Local run on port 3000. |
| [.github/workflows/deploy.yml](../../.github/workflows/deploy.yml) | Pipeline (repo root; `DEPLOY_ROOT` points here). |
| [DEPLOYMENT.md](DEPLOYMENT.md) | EC2, security group, secrets, issues we hit, and bonus rollback. |

---

## Run locally

```bash
cd app && npm install && npm test
```

With Docker (from this directory):

```bash
cp .env.example .env
docker compose up --build
```

Open `http://localhost:3000/health` — expect `{"status":"ok"}`.

---

## Cloud checklist (before you submit)

- [ ] `docker compose up --build` works locally; `.env` is **not** committed.
- [ ] GitHub **Actions** shows a **green** run on `main` (test → build+push → deploy).
- [ ] `GET http://<ec2-public-ip>/health` returns **200** and JSON with `"ok"`.
- [ ] **SSH** in the security group is **not** `0.0.0.0/0` (HTTP **may** be `0.0.0.0/0`).
- [ ] No `.pem` or API tokens in Git — only in **GitHub → Settings → Secrets**.
- [ ] [DEPLOYMENT.md](DEPLOYMENT.md) is filled in for your environment.

**Submission:** submit your fork’s URL via the [AmaliTech form](https://forms.cloud.microsoft.com/e/f3FF83LVz3) as required by the program.

---

## Glossary (quick)

| Term | Meaning |
|------|--------|
| **CI** | Automated test/build on every change (here: GitHub Actions on `main`). |
| **CD** | Delivering the build to a server (here: pull new image on EC2 and restart the container). |
| **CIDR** | IP range used in firewall rules, e.g. `203.0.113.0/32` for a single address. |
| **GHCR** | GitHub’s container registry; image URL `ghcr.io/<owner>/<repo>/...`. |
| **Security group** | EC2’s inbound/outbound access rules. |

For longer explanations and the problems we hit in practice, see [DEPLOYMENT.md](DEPLOYMENT.md).
