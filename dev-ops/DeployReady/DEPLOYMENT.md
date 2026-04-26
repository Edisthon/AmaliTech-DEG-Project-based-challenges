# Kora API — AWS deployment and pipeline

This file documents the DeployReady submission: what was provisioned, how CI/CD works, and what we hit along the way. Replace any `<placeholders>` with your own values when you fork this.

---

## 1. What we built (high level)

| Piece | Role |
|--------|------|
| **Dockerfile** | Multi-stage not required; `node:20-alpine`, non-root `node`, `PORT` for the app. |
| **docker-compose** | Local dev: map host `3000` to the container, `PORT` from `.env`. |
| **GitHub Actions** | On push to `main`: `npm test` → build+push image to **GHCR** (tag = commit SHA) → **SSH to EC2** → `docker pull` + restart `kora` → **GET `/health` on the instance**; on failure, **roll back** to the image that was running before (bonus). |
| **EC2** | Amazon Linux 2023, Docker, single container on host port `80` → app `3000`. |

**Terms worth knowing**

- **GHCR (GitHub Container Registry)** — OCI registry tied to GitHub; the workflow authenticates with `GITHUB_TOKEN` to push, and the server pulls with a PAT (if the package is private) or without extra login (if the package is public).
- **Security group** — EC2’s stateful firewall: which ports accept traffic from which **CIDRs** (IP ranges). HTTP `80` from `0.0.0.0/0` is normal for a public API; SSH `22` should **not** be `0.0.0.0/0` for this challenge (use **your IP** and/or the **`actions` ranges** from [GitHub’s meta API](https://api.github.com/meta) so the runner can SSH).
- **Immutability** — Image tag = `git` commit SHA, so what runs in production is traceable to a exact commit.
- **Rollback** — After deploy, `curl` to `http://127.0.0.1/health` on the box. If that fails, the script stops the new container and starts the **previous** image again (if there was one). The workflow **still fails** (red) so the team knows the new build did not go live healthy.

---

## 2. EC2 instance

- **Region:** e.g. `eu-west-1` (your choice).
- **AMI:** Amazon Linux 2023.
- **Instance type:** `t2.micro` (or `t3.micro` in regions without `t2`).
- **Key pair:** create in EC2, download the `.pem`, **never** commit it. Add only the public key to the instance.
- **Public IPv4:** enable so `http://<public-ip>/health` works for the grader and for manual checks.
- **IAM on the host:** optional instance profile. This pipeline does **not** need AWS access keys in GitHub for deploy (image path is **GHCR**, not ECR). Document what you attached if you added a role: **&lt;your note&gt;**

---

## 3. Security group

| Type | Port | Source | Why |
|------|------|--------|-----|
| HTTP | 80 | `0.0.0.0/0` | Public access to the API and `/health`. |
| SSH | 22 | **Not** `0.0.0.0/0` | Per challenge: restrict to your IP, office VPN, and/or the **`actions`** CIDRs from [api.github.com/meta](https://api.github.com/meta) so **GitHub Actions** can run `appleboy/ssh-action`. |

Outbound: default “allow all” is fine (pull from `ghcr.io`, DNS, etc.).

---

## 4. IAM “for the pipeline” (how we meet the requirement)

- **Pushing the image** uses **GitHub** (`GITHUB_TOKEN` with `packages: write` in the workflow), not an IAM user in AWS.
- **Satisfying “IAM user or role with least privilege”** — e.g. document an **EC2 instance role** for future SSM/CloudWatch, or a **break-glass IAM user** for operators, with no long-lived keys in the repo. We did **not** store AWS access keys in GitHub for this challenge path.

**What we used:** &lt;briefly: e.g. “instance profile for optional SSM” or “no extra IAM beyond default account practices”&gt;

---

## 5. Install Docker (Amazon Linux 2023)

```bash
sudo dnf update -y
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
```

The deploy script uses `sudo docker` so it works even before you add `ec2-user` to the `docker` group.

---

## 6. GitHub repository secrets (names only; values are never committed)

| Secret | Purpose |
|--------|---------|
| `EC2_HOST` | Public DNS or IP of the instance. |
| `EC2_USER` | e.g. `ec2-user`. |
| `EC2_SSH_KEY` | Private key PEM (full multiline). |
| `GHCR_PAT` / `GHCR_USER` | Optional, if the GHCR image is **private** — the EC2 `docker pull` path logs in. |

**Package visibility:** if the **kora-api** package is **public** on `ghcr.io`, you can leave `GHCR_PAT` empty in practice once pull works without auth.

---

## 7. First manual run (optional)

```bash
export IMAGE=ghcr.io/<owner-lowercase>/<repo>/kora-api:<commit-sha>
sudo docker pull "$IMAGE"
sudo docker run -d --name kora --restart unless-stopped -p 80:3000 -e PORT=3000 "$IMAGE"
```

After that, every green push to `main` runs the same lifecycle via the workflow.

---

## 8. Verify the app

**On the host**

```bash
sudo docker ps --filter name=kora
```

Expect `0.0.0.0:80->3000/tcp` (or similar).

**From a browser or laptop**

`http://<EC2_PUBLIC_IP>/health` → `{"status":"ok"}`

**Logs**

```bash
sudo docker logs kora
sudo docker logs -f kora
```

---

## 9. Bonus: rollback in the pipeline

**Implemented:** the remote deploy script saves the **current** image ref (`OLD_IMAGE`) before stopping `kora`, starts the new image, then runs:

`curl -sf http://127.0.0.1/health`

If that fails (no `200` / connection error), it removes the new container and runs `OLD_IMAGE` again **if** it was set. The GitHub job **exits with failure** so the failed release is still visible in Actions even if the service was rolled back to the last good image.

### 9.1 How we tested rollback (what we did)

**Goal:** prove the pipeline fails a bad image and **restores the previous** running image, without teaching CI to deploy broken application code (tests would block that if we only changed `index.js`).

| Step | What we did |
|------|----------------|
| 1 | **Pre-requisite:** a **prior successful deploy** on EC2 (container `kora` with a good image) so the script can capture a real `OLD_IMAGE`. |
| 2 | **Change only the `Dockerfile`** for one commit — e.g. replace the final `CMD` with something that **keeps the container up** but **does not** run `node` on port 3000 (so nothing answers on `http://127.0.0.1:80` / `/health`). **Do not** change `app/index.js` for this: `npm test` uses the source in the repo, so the **Test** job stays green while the **built image** is intentionally broken. |
| 3 | **Push to `main`.** The workflow: builds and pushes a new image (new commit SHA) → **Deploy** pulls and starts it → post-deploy `curl` fails. |
| 4 | **Expected in the “Deploy to EC2” log:** `POST_DEPLOY_HEALTH_FAILED: rolling back to previous image if it existed.` then a line like `Rollback to ghcr.io/<owner>/<repo>/kora-api:<previous-git-sha> started; check EC2. Deploy step still fails so CI reports the bad release.` The job ends **red** (exit 1) **on purpose** — a failed release is still reported. |
| 5 | **Restore:** revert the `Dockerfile` to the real `USER node` + `CMD ["node", "index.js"]`, commit, push. The next run should be **green**; the new good image supersedes the bad one. |

**Why Dockerfile-only?** If you break `GET /health` in `index.js` alone, **`npm test`** (supertest) will usually fail, so the pipeline **never** reaches deploy. The Dockerfile is what defines the **runtime** image, so you can make the **container** fail while tests still pass.

### 9.2 What this proved

- A bad deploy is **caught** by a **post-deploy** check on the instance (not only by pre-deploy tests).
- The service can be **rolled back** to the last running image so users still get a working `/health` (verify with the browser to the public IP after the failed run).
- **GitHub Actions** still shows a **failed** run for the bad commit, which is the right signal for a team to investigate.

---

## 10. Challenges and how we resolved them (learning notes)

| Symptom | Likely cause | What worked |
|--------|----------------|------------|
| `dial tcp …:22: i/o timeout` from Actions | Security group does not allow the **runner’s** IP | Add **`actions` CIDRs** from GitHub’s meta API (or a temporary — **then tighten**; challenge expects SSH **not** `0.0.0.0/0` at submit time). |
| `unknown blob` on `docker push` to GHCR | BuildKit attestations + separate push | `docker buildx build --provenance=false --sbom=false --push` in one step. |
| `ssh: no key found` in Actions | Malformed or truncated `EC2_SSH_KEY` secret | Full PEM in the secret, one block. |
| `docker ps` empty on EC2 | No successful deploy / wrong image / pull auth | Fix pipeline, ensure GHCR pull works, check `EC2_HOST`. |

---

## 11. Summary for reviewers

| Item | Your value |
|------|------------|
| Public URL (health) | `http://&lt;EC2_IP_OR_DNS&gt;/health` |
| Image pattern | `ghcr.io/.../kora-api:&lt;git-sha&gt;` |
| Rollback | Yes — post-deploy `curl` to `/health`, revert to previous image on failure. |
| Rollback tested | Yes — via temporary **Dockerfile-only** bad `CMD`; observed `POST_DEPLOY_HEALTH_FAILED` + rollback log + red job; then **reverted** Dockerfile (see §9.1). |

---

*Keep this file in the repository (commit and push). It is documentation, not a secret. Do not put `.pem` files or raw tokens in here.*
