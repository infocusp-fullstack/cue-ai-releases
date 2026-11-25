# Cue AI — desktop releases

Download the Cue AI desktop app from the **[Releases](../../releases)** page.

| Platform | File |
| --- | --- |
| macOS (Apple silicon) | `cue-ai-<version>-mac-arm64.dmg` (or `.zip`) |
| Windows | `cue-ai-<version>-win-x64.exe` |
| Linux | `cue-ai-<version>-linux-x86_64.AppImage` or `cue-ai-<version>-linux-amd64.deb` |

Releases whose tags end in `-qa.<n>` are **pre-releases** for testing. Every release has a `SHA256SUMS.txt` file for checking the downloads.

The installers are not code-signed yet:

- **macOS:** right-click the app → *Open* the first time. If macOS says the app "is damaged", run
  `xattr -cr "/Applications/Cue AI.app"` once in Terminal (this removes the download quarantine flag).
- **Windows:** *More info → Run anyway*.

## About this repository

This repository contains **only** the build workflow (`.github/workflows/release.yml`) and this README.
The application source code is private. Each workflow run clones the source with a read-only token, builds the installers on GitHub-hosted runners and publishes them here.

Only maintainers can start a build (**Actions → Release desktop app → Run workflow**).

| Secret | Required | Purpose |
| --- | --- | --- |
| `PRIVATE_REPO_PAT` | yes | fine-grained token, read-only `Contents` on the private source repo only |
| `CSC_LINK`, `CSC_KEY_PASSWORD` | no | macOS signing certificate |
| `WIN_CSC_LINK`, `WIN_CSC_KEY_PASSWORD` | no | Windows signing certificate |

Do not edit `release.yml` here directly. Its master copy is in the private source repo (`docs/release-workflow/release.yml`), which also has the full setup guide.
