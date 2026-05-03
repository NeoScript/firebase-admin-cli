# Releasing fbadmin

Releases are automated via [cargo-dist](https://opensource.axo.dev/cargo-dist/). Pushing a semver tag triggers GitHub Actions which builds binaries, creates a GitHub Release, and publishes a Homebrew formula.

## Prerequisites

- Push access to `NeoScript/firebase-admin-cli`
- `HOMEBREW_TAP_TOKEN` repo secret set on `firebase-admin-cli` — a fine-grained PAT with **Contents: Read and write** on `NeoScript/homebrew-fbadmin`

## Release checklist

1. **Bump the version** in `Cargo.toml`:

   ```toml
   [package]
   version = "0.2.0"
   ```

2. **Update RELEASES.md** — add a section for the new version at the top:

   ```markdown
   # v0.2.0

   - Added feature X
   - Fixed bug Y
   ```

3. **Regenerate the release workflow** (only needed if you changed `dist-workspace.toml`):

   ```bash
   dist generate
   ```

4. **Commit**:

   ```bash
   git add -A
   git commit -m "chore: prepare v0.2.0 release"
   ```

5. **Tag and push**:

   ```bash
   git tag v0.2.0
   git push origin main --tags
   ```

6. **Monitor** the Release workflow at https://github.com/NeoScript/firebase-admin-cli/actions

## What the workflow does

The `release.yml` workflow:

1. **plan** — determines what to build from the tag
2. **build-local-artifacts** — compiles binaries for each target:
   - `aarch64-apple-darwin` (Apple Silicon)
   - `x86_64-apple-darwin` (Intel Mac)
   - `aarch64-unknown-linux-gnu` (Linux ARM64)
   - `x86_64-unknown-linux-gnu` (Linux x64)
   - `x86_64-pc-windows-msvc` (Windows x64)
3. **build-global-artifacts** — generates shell/powershell installer scripts and checksums
4. **host** — creates the GitHub Release and uploads all artifacts
5. **publish-homebrew-formula** — pushes a `.rb` formula to `NeoScript/homebrew-fbadmin`
6. **announce** — finalizes the release

## Configuration

Release settings live in `dist-workspace.toml`:

```toml
[dist]
cargo-dist-version = "0.31.0"
ci = "github"
installers = ["shell", "powershell", "homebrew"]
targets = [...]
tap = "NeoScript/homebrew-fbadmin"
publish-jobs = ["homebrew"]
```

To change targets, installers, or upgrade cargo-dist, edit this file and run `dist generate` to regenerate the workflow.

## Adding a new target

1. Add the Rust target triple to `targets` in `dist-workspace.toml`
2. Run `dist generate`
3. Commit the updated `dist-workspace.toml` and `.github/workflows/release.yml`

## Troubleshooting

- **Workflow didn't trigger**: the workflow must already exist on the branch when the tag is pushed. If you added the workflow and tag in the same push, delete and re-push the tag:

  ```bash
  git push origin :refs/tags/vX.Y.Z
  git tag -d vX.Y.Z
  git tag vX.Y.Z
  git push origin vX.Y.Z
  ```

- **Homebrew publish failed**: verify the `HOMEBREW_TAP_TOKEN` secret is set and the PAT hasn't expired. The token needs **Contents: Read and write** on `NeoScript/homebrew-fbadmin`.

- **Build failed for a target**: check the build logs in GitHub Actions. Common causes are missing system deps for cross-compilation or Rust version mismatches.
