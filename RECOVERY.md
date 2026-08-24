# RECOVERY.md — spekt8

**Rebuild difficulty:** EASY for the code, BLOCKED for automated publishing.
Everything tracked is public and already on GitHub, and the released image is
live on Docker Hub. But the Docker Hub push credential **has already expired** —
see Part 1 → "What you need that isn't in the repo". Nothing about the SSD swap
caused that; it is broken right now.

Verified against the repo, the GitHub API, Docker Hub and the local filesystem
on **2026-08-23 20:25 PDT**. Every claim below was checked; nothing needed to be
left UNKNOWN. Figures are measured, not estimated.

---

# Part 1 — Recovery strategy

## What this is
A React/Redux + Express web app that reads the Kubernetes API and draws a live
topology of pods, services and ingresses in the `default` namespace. It is
**entirely stateless** — it stores nothing, writes nothing, and holds no data of
its own; every view is fetched from whatever cluster it is running in. There is
no database, no volume, no cache: nothing to dump before a copy.

Origin is `https://github.com/spekt8/spekt8.git` — public, not a fork, and the
`gh` account `syntheticproduct` has ADMIN on it. Note this repo is under the
`spekt8` org, **not** under `syntheticproduct` like the rest of my projects.

## Rebuild from scratch
1. `gh repo clone spekt8/spekt8 && cd spekt8`
2. Install Node 20 (image uses `node:20-alpine`). **Node 24 is verified working
   locally** — the build was run on v24.15.0 / npm 12.0.1 during this check.
3. `npm install` — there is **no committed lockfile**, see Gotchas.
4. `npm run build` (writes `dist/main.js`) then `npm run server`
   → http://localhost:3000
   Verified: webpack 5.106.2 compiles in ~8s with 4 size/deprecation warnings,
   no errors. **Building dirties the tracked tree — see Gotchas before you
   commit anything afterwards.**
5. ~~Tests: `npm test`~~ — **`npm test` is broken and does not run.** Both suites
   fail to load with `Cannot find module 'cheerio/lib/utils'` from enzyme;
   0 tests execute. See Gotchas. Do not use it as a rebuild checkpoint.
6. Container build: `docker build -t spekt8 . && docker run -p 3000:3000 spekt8`
   (Dockerfile is a two-stage `node:20-alpine` build, runs as `USER node`.)
7. To restore automated publishing, reissue and set two repo secrets:
   `gh secret set DOCKERHUB_USERNAME --repo spekt8/spekt8` and
   `gh secret set DOCKERHUB_TOKEN --repo spekt8/spekt8` (new access token from
   https://hub.docker.com → Account Settings → Personal access tokens).
   **This is required today, not just after a rebuild** — the current token is
   expired and CI is red.
8. Deploy to a cluster: `kubectl apply -f spekt8-rbac.yaml`, then
   `kubectl apply -f spekt8-deployment.yaml`, then
   `kubectl port-forward deployment/spekt8 3000:3000`.
   Apply the RBAC first — the Deployment references the `spekt8` ServiceAccount.
9. **Verify:** open http://localhost:3000 and confirm pods render in graph view.
   `curl localhost:3000/pod` must return a **pod list**. Checking the HTTP status
   is not enough: with no cluster reachable the endpoint still answers **200**
   with an error object body (`{"errno":-111,"code":"ECONNREFUSED",...,"port":8080}`)
   — measured. Read the body, not the code.

## What you need that isn't in the repo
| what | where it lives now | how to get it again |
|---|---|---|
| `DOCKERHUB_TOKEN` | GitHub Actions secret on spekt8/spekt8, set 2026-05-11 00:23 UTC. **EXPIRED — CI run 32680085794 on 2026-08-24 01:32 UTC failed at "Log in to Docker Hub": `unauthorized: personal access token is expired`.** | Reissue at hub.docker.com → personal access tokens. Not readable back. |
| `DOCKERHUB_USERNAME` | same, set 2026-05-11 00:24 UTC | Docker Hub account `camillelambert` (personal account, not the spekt8 org). Pushes to `camillelambert/spekt8`. |
| GitHub push rights | `gh` CLI as `syntheticproduct`, ADMIN on spekt8/spekt8, HTTPS, scopes incl. `repo` + `workflow` | `gh auth login` over HTTPS. |
| `package-lock.json` (772 KB) | **local only — gitignored, untracked.** Generated 2026-05-10. | Cannot be regenerated identically; `npm install` today resolves differently. See Gotchas + Part 2. |
| `node_modules` (377 MB) | untracked, local only | `npm install`. |
| `kubectl` | **not installed on this machine** (`docker` is) | `apt install kubectl` or equivalent. |
| a Kubernetes cluster | none on this machine — no `~/.kube` exists | Any cluster; the app only needs read access in `default`. |

## Has no copy anywhere
**Almost nothing — but not literally nothing.**

Verified covered: working tree is clean; `master` is byte-identical to
`origin/master` (`b83f930`); tag `v2.0.0` is on the remote; `git log --all
--not --remotes` is empty, so every branch and commit exists on GitHub; there is
no stash. The published multi-arch image (`linux/amd64`, `linux/arm64`,
`linux/arm/v7`) is confirmed live on Docker Hub, pushed 2026-05-11, 4 tags,
public, 420 pulls.

Not covered by the remote, because git does not track them:
- **`package-lock.json`** — gitignored. It exists inside the WSL disk image
  captured to `D:\_backup\wsl` at 13:05 on 2026-08-23, so it is not lost, but
  the only copy outside this machine is buried in a 122 GB `.vhdx`.
- **This file, as rewritten** — the corrections below were written after that
  13:05 snapshot and are deliberately not committed. See Part 2.

## Gotchas
- **`package-lock.json` is in `.gitignore`.** Neither a fresh clone nor the
  Dockerfile (`npm install`, not `npm ci`) pins versions, and the dependency set
  is 2018-era with caret ranges throughout. A rebuild years from now may resolve
  to incompatible transitive versions. **This has already happened once** — see
  the test failure below.
- **`npm test` is dead, and restoring the lockfile will not revive it.** enzyme
  (unmaintained, last real release 2020) does `require('cheerio/lib/utils')`,
  a path cheerio dropped. The resolved cheerio is **1.2.0**, and that version is
  what the local `package-lock.json` itself pins — so this is not drift you can
  roll back, it was baked in at the 2026-05-10 install. Fixing it means dropping
  enzyme for React Testing Library. Until then the repo has **zero** working
  tests despite shipping two test files.
- **`npm run build` dirties the tracked working tree.** `dist/` is committed, but
  the modernized webpack 5 emits assets to different paths than the 2018 bundle:
  images move from `dist/images/src/client/images/` to `dist/images/`, and a new
  `dist/main.js.LICENSE.txt` appears. After a local build `git status` shows ~10
  deletions, 2 modifications and 10 untracked files. Don't mistake that for real
  work and commit it — `git checkout -- dist/ && git clean -fd dist/` restores.
- **The k8s client is `^0.7.1` (0.7.2 installed) and calls `Extensions_v1beta1Api`.**
  The symbol still exists client-side, so the server boots fine — but the
  `extensions/v1beta1` API *group* was removed in Kubernetes 1.22, so `/ingress`,
  `/deployment` and `/daemonset` will fail against any modern cluster; `/pod` and
  `/service` (core v1) still work. Fixing this means upgrading
  `@kubernetes/client-node` and moving to `networking.k8s.io/v1` + `apps/v1`.
- **`spekt8-rbac.yaml` grants the same dead API group.** Its Role rule for
  ingresses/deployments/daemonsets is scoped to `apiGroups: ["extensions"]`, so
  on k8s ≥1.22 that rule binds to nothing. It must be migrated in lockstep with
  the client above, or the endpoints will 403 even after the code is fixed.
- **Every endpoint returns HTTP 200 on failure.** The error handlers are
  `.catch(err => res.send(err))` — no status is set, so a health check or smoke
  test that only asserts `200` will pass against a completely broken app.
- **Port 3000 is hardcoded** in `src/server/server.js` with no `PORT` env
  override, and **something else is already listening on 3000 on this machine**
  (a Next.js app). Starting spekt8 here silently gives you the other app's
  responses. Free the port, or edit the literal.
- `dist/main.js` is committed but last changed in commit `e26bd24`, **2018-11-20**.
  The Docker build regenerates it, so the image is fine — don't trust the
  checked-in bundle.
- Four `.DS_Store` files are tracked (`k8sed/`, `src/`, `src/assets/`,
  `src/client/`); harmless, but they are why the `.gitignore` entry looks
  ineffective — ignoring a path does not untrack it.
- This project has **no `CLAUDE.md` and no `.claude/` directory**. Everything
  needed is in the repo; nothing important lives only in session transcripts.

---

# Part 2 — If C: is wiped tomorrow

## Tactical — copy before the wipe

**Everything this project has ever pushed is already safe off-machine** — all
commits, branches and the `v2.0.0` tag are on GitHub, and the released
multi-arch image is on Docker Hub. Neither depends on this SSD.

**No dump step applies.** The app is stateless — no database, no volume, no
on-disk state — so nothing has to be quiesced before copying.

Not listed below, because they are already covered: the git history and all
tracked files (on GitHub), and everything under `/home/camille` as it stood at
2026-08-23 13:05, which is inside `D:\_backup\wsl\Ubuntu\ext4.vhdx` per the
manifest. `node_modules` (377 MB) is not listed — it is reproducible with
`npm install` and is in that image anyway.

Two files are worth 780 KB of insurance anyway:

| what | source path (on C:) | size | destination | already covered? |
|---|---|---|---|---|
| This corrected `RECOVERY.md` — rewritten 2026-08-23 ~20:30, deliberately **not** committed, so it is in neither git nor the 13:05 disk image | `/home/camille/projects/spekt8/RECOVERY.md` | 10.5 KB | `D:\_backup\projects\spekt8\` | **No — no copy anywhere** |
| `package-lock.json` — gitignored, the only pinned resolution of a 2018-era dep tree that still installs | `/home/camille/projects/spekt8/package-lock.json` | 772 KB | `D:\_backup\projects\spekt8\` | Yes, but only *inside* the 122 GB `.vhdx` — this makes it a single-file restore |

**Copy now:**

```
mkdir -p /mnt/d/_backup/projects/spekt8
cp /home/camille/projects/spekt8/RECOVERY.md       /mnt/d/_backup/projects/spekt8/
cp /home/camille/projects/spekt8/package-lock.json /mnt/d/_backup/projects/spekt8/
```

Plain `cp` of two files into a new subdirectory — no mirroring, no deletion.
Never run `robocopy /MIR`, `/PURGE`, or `rsync --delete` against `D:\_backup`:
it is a shared destination holding other jobs' only copies.

Better than any copy, if you are willing to commit: `git add -f package-lock.json
RECOVERY.md && git commit && git push` puts both somewhere that survives D:
failing as well as C: being wiped. The copy above is the do-not-commit option.

TOTAL: 782 KB, under 1 minute.
