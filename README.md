# Release workflow (public build repo + private source repo)

Cue AI's desktop installers (macOS, Windows, Linux) are built by GitHub Actions in the **public** repository
[`infocusp-fullstack/cue-ai-releases`](https://github.com/infocusp-fullstack/cue-ai-releases).
Builds there run on the free Actions minutes for public repositories. The source code stays in this **private** repository.

| Repository | Visibility | Contains |
| --- | --- | --- |
| `infocusp-fullstack/cue-ai` (this one) | private | all source code; this folder holds the master copy of the workflow |
| `infocusp-fullstack/cue-ai-releases` | public | only `.github/workflows/release.yml`, a README, and the Releases page with the installers |

Each run clones this repo with a read-only token, runs the checks, builds the installers and attaches them to a GitHub Release in the public repo.

> **Visibility note:** the Releases page of `cue-ai-releases` is **public**. Anyone on the internet can download the installers.
> The source code stays private. Use the `draft` option if you want to review a release before it becomes visible.

Files in this folder:

- [`release.yml`](release.yml): the workflow. **Edit it here**, then copy it to the public repo.
- [`public-repo-README.md`](public-repo-README.md): the README to put in the public repo.
- Day-to-day usage (running a build, rotating the token): [`../building-installers.md`](../building-installers.md).

---

## One-time setup

The public repo already exists (created 2026-11-23). A person with admin rights on it does the steps below. The agent that wrote this cannot push to that repo.

### 1. Put the workflow and README in the public repo

```bash
git clone https://github.com/infocusp-fullstack/cue-ai-releases.git
cd cue-ai-releases
mkdir -p .github/workflows
cp ../cue-ai/docs/release-workflow/release.yml .github/workflows/release.yml
cp ../cue-ai/docs/release-workflow/public-repo-README.md README.md
git add .github/workflows/release.yml README.md
git commit -m "Add desktop release workflow"
git push
```

Copy **only** these two files. Do not copy any source code, `.env`, or history from the private repo.

### 2. Create the read-only token (`PRIVATE_REPO_PAT`)

1. GitHub → your avatar → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.
2. **Token name:** `cue-ai-releases read source`.
3. **Expiration:** 90 days (put a reminder in your calendar a week before it expires).
4. **Resource owner:** `infocusp-fullstack`. If the organisation requires approval for fine-grained tokens, an org owner has to approve it.
5. **Repository access:** *Only select repositories* → `infocusp-fullstack/cue-ai`. Select only this repo.
6. **Permissions → Repository permissions → Contents: Read-only**. `Metadata: Read-only` is added automatically. Leave everything else at *No access*.
7. Generate the token and copy it. GitHub shows it only once.

### 3. Store the token as a secret

Use one of these options. Never put the token in a file.

- **Repository secret (simplest):** `cue-ai-releases` → **Settings → Secrets and variables → Actions → New repository secret**.
  Name: `PRIVATE_REPO_PAT`. Value: the token.
- **Organisation secret:** org **Settings → Secrets and variables → Actions → New organization secret**.
  Name: `PRIVATE_REPO_PAT`. Repository access: *Selected repositories* → only `cue-ai-releases`.

**Optional signing secrets.** Store them the same way. Builds work **unsigned** without them; signing and notarization are not set up yet.

| Secret name | Used for |
| --- | --- |
| `CSC_LINK` | macOS signing certificate (.p12, base64) |
| `CSC_KEY_PASSWORD` | password of `CSC_LINK` |
| `WIN_CSC_LINK` | Windows signing certificate (.pfx, base64) |
| `WIN_CSC_KEY_PASSWORD` | password of `WIN_CSC_LINK` |

### 4. Check the repo settings

- **Settings → Actions → General → Workflow permissions:** *Read repository contents and packages permissions*.
  The workflow asks for `contents: write` itself, and only in the publish job.
- Keep the list of people with **write** access to `cue-ai-releases` short. Anyone with write access can start a build, and every build uses the token.

### 5. Run the first build

1. `cue-ai-releases` → **Actions → Release desktop app → Run workflow**.
2. Pick `branch: main`, keep all three platforms selected, and tick **draft** for the first run.
3. Wait about 15–25 minutes. The run shows three stages: **Checks → Build (mac / win / linux) → Publish GitHub Release**.
4. Open **Releases**. You will see a draft `v1.0.0` with these files:
   `cue-ai-1.0.0-mac-arm64.dmg`, `cue-ai-1.0.0-mac-arm64.zip`, `cue-ai-1.0.0-win-x64.exe`,
   `cue-ai-1.0.0-linux-x86_64.AppImage`, `cue-ai-1.0.0-linux-amd64.deb`, `SHA256SUMS.txt`.
5. Download one installer and try it. If it is good, click **Edit → Publish release** to make it public.

---

## How the workflow works

| Step | Job | What happens |
| --- | --- | --- |
| 1 | `verify` (Linux) | Clones the chosen branch, reads the version from `electron/package.json`, works out the tag, reads the Node version from `electron/.nvmrc`, then runs `npm ci`, `typecheck`, `lint` and the unit tests. If any check fails, nothing is built. |
| 2 | `build` (macOS, Windows, Linux in parallel) | Clones the **same commit** again, then runs `npm ci`, `npm run build`, `npm run build:electron` and `npm run package -- --<platform>` (electron-builder). Uploads the installers to the run. |
| 3 | `publish` (Linux) | Runs only if every selected platform built. Creates the GitHub Release, uploads all installers and adds `SHA256SUMS.txt`. |

**Inputs** (asked for when you click *Run workflow*):

| Input | Default | Meaning |
| --- | --- | --- |
| `branch` | `main` | `main` makes a normal release; `qa` makes a pre-release |
| `build_macos` / `build_windows` / `build_linux` | all on | which platforms to build |
| `draft` | off | create the release as a draft (hidden until you publish it) |

**Tags.** The workflow never triggers on schedule or push; it starts only manually. Tags are:

- `main` → `v<version>`, for example `v1.0.0`. Running `main` twice with the same version fails on purpose. Bump `version` in `electron/package.json` first.
- `qa` → `v<version>-qa.<run number>`, for example `v1.0.0-qa.17`, marked as **pre-release**.

**How the token is protected:**

- It is passed to the shell as a masked environment variable. The script never echoes it and never uses `set -x`.
- After cloning, `git remote remove origin` deletes the URL that contains the token from `.git/config`, and the step fails if the token is still there.
- Only the `publish` job can write to the public repo, and it uses the built-in `GITHUB_TOKEN`, not the PAT.
- Logs list installer **names and sizes** only, never file contents.

---

## Building on your own machine

The same commands run locally through [`scripts/build-local.sh`](../../scripts/build-local.sh):

```bash
scripts/build-local.sh                       # build for this machine's OS
scripts/build-local.sh linux --dir ~/cue-out # Linux installers into ~/cue-out
scripts/build-local.sh all                   # every platform this machine can build
scripts/build-local.sh win --dry-run         # only print the commands
```

See [`../building-installers.md`](../building-installers.md#build-on-your-own-machine) for details.
