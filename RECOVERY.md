# RECOVERY.md — spekt8

**Rebuild difficulty:** MODERATE
(Code and image are fully public and reproducible from the remote. The only thing
that can't be cloned back is the Docker Hub push credential in GitHub Actions.)

## What this is
A React/Redux + Express web app that reads the Kubernetes API and draws a live
topology of pods, services and ingresses in the `default` namespace. It is
**entirely stateless** — it stores nothing, writes nothing, and holds no data of
its own; every view is fetched from whatever cluster it is running in.

## Rebuild from scratch
1. `gh repo clone spekt8/spekt8 && cd spekt8`
2. Install Node 20 (image uses `node:20-alpine`; Node 24 works locally).
3. `npm install` — there is **no committed lockfile**, see Gotchas.
4. `npm run build` (writes `dist/main.js`) then `npm run server` → http://localhost:3000
5. Tests: `npm test` (jest + enzyme).
6. Container build: `docker build -t spekt8 . && docker run -p 3000:3000 spekt8`
7. To restore automated publishing, set two repo secrets in GitHub Actions:
   `gh secret set DOCKERHUB_USERNAME --repo spekt8/spekt8` and
   `gh secret set DOCKERHUB_TOKEN --repo spekt8/spekt8` (new access token from
   https://hub.docker.com → Account Settings → Personal access tokens).
8. Deploy to a cluster: `kubectl apply -f spekt8-rbac.yaml`, then
   `kubectl apply -f spekt8-deployment.yaml`, then
   `kubectl port-forward deployment/spekt8 3000:3000`.
9. **Verify:** open http://localhost:3000 and confirm pods render in graph view;
   `curl localhost:3000/pod` should return JSON, not an error object.

## What you need that isn't in the repo
| what | where it lives now | how to get it again |
|---|---|---|
| `DOCKERHUB_TOKEN` | GitHub Actions secret on spekt8/spekt8 (set 2026-05-11) | Reissue at hub.docker.com → personal access tokens. Not readable back. |
| `DOCKERHUB_USERNAME` | same | Docker Hub account `camillelambert` (personal account, not the spekt8 org). |
| GitHub push rights | `gh` CLI as `syntheticproduct`, ADMIN on spekt8/spekt8 | `gh auth login` over HTTPS. |
| `node_modules` (377 MB) | untracked, local only | `npm install`. |
| `kubectl` | **not installed on this machine** | `apt install kubectl` or equivalent. |
| a Kubernetes cluster | none on this machine — no `~/.kube` exists | Any cluster; the app only needs read access in `default`. |

## Has no copy anywhere
Nothing. Working tree is clean, `master` is level with `origin/master`, all
feature branches are pushed, and the published multi-arch image (amd64, arm64,
arm/v7) is live on Docker Hub. Losing this machine loses nothing but `node_modules`.

## Gotchas
- **`package-lock.json` is in `.gitignore`.** Neither a fresh clone nor the
  Dockerfile (`npm install`, not `npm ci`) pins versions, and the dependency set
  is 2018-era with caret ranges. A rebuild years from now may resolve to
  incompatible transitive versions. If a fresh `npm install` fails, that's why.
- **The k8s client is pinned to 0.7.1 and calls `Extensions_v1beta1Api`.** The
  `extensions/v1beta1` group was removed in Kubernetes 1.22, so `/ingress`,
  `/deployment` and `/daemonset` will fail against any modern cluster; `/pod` and
  `/service` (core v1) still work. Fixing this means upgrading
  `@kubernetes/client-node` and moving to `networking.k8s.io/v1` + `apps/v1`.
- `dist/` is committed but the tracked `dist/main.js` dates from 2018. The Docker
  build regenerates it, so the image is fine — don't trust the checked-in bundle.
- Four `.DS_Store` files are tracked in git; harmless, but they are why the
  `.gitignore` entry looks ineffective.
- This project has **no `CLAUDE.md` and no `.claude/` directory**. Everything
  needed is in the repo; nothing important lives only in session transcripts.
