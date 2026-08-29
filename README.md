# pkgs/packaging — shared build + supply-chain pipeline

Every package repo under the `pkgs` org uses one of the reusable workflows here,
so the security gates are defined **once** and can't be weakened per-repo.

## Add a new package

1. New repo `pkgs/<name>` (push-to-create, or create in the UI).
2. Copy `templates/` into it and adjust:
   - `PKGBUILD` — pkgname, deps, build/package steps
   - `deny.toml`, `.semgrep.yml`, `supply-chain/` — usually unchanged
   - `renovate.json` — unchanged
   - `.gitea/workflows/build.yml` — set `pkgname`, pick `rust-pkg` or `generic-pkg`
3. `makepkg --printsrcinfo > .SRCINFO`, commit, push.
4. Cut a release: `./release.sh 0.1.0` (bumps versions, tags; the tag push
   builds + publishes to the Arch registry).

The `/clockmost-package` Claude skill automates 1–4.

## Reusable workflows

| workflow | for | jobs |
|---|---|---|
| `rust-pkg.yml` | Cargo projects | `scan` (fmt, clippy, cargo-deny, cargo-audit, **cargo-vet**, semgrep, build, trivy fs, cargo-geiger) → `package` (tag only: makepkg, trivy rootfs, namcap, publish, release) |
| `generic-pkg.yml` | scripts / prebuilt | `package` (makepkg, trivy rootfs, namcap, publish, release) |

`scan` runs on every push + PR and blocks merge. `package` runs only on a `v*`
tag and blocks publish.

## cargo-vet fails the build — what to do

`cargo vet` blocks when a dependency (new, or a version bump) isn't covered by an
imported audit set. **Resolve it with a Claude session, not by hand:**

```
cargo vet suggest                       # what needs auditing (also printed in CI)
cargo vet inspect <crate> <version>     # new crate — read the source
cargo vet diff <crate> <old> <new>      # bump — read only the delta
```

Review for: build scripts, proc-macros, `unsafe`, process/network/fs calls,
obfuscation, anything off-purpose. Then record it:

```
cargo vet certify <crate> <version>     # --criteria safe-to-deploy (default)
git add supply-chain/audits.toml && git commit -m "vet: <crate> <version>" && git push
```

Imported audit sets: mozilla, google, bytecodealliance, embark-studios, isrg, zcash.

## Client (pacman)

See `~/runbooks/clockmost-packages.md` on the clockmost host, or the
`/clockmost-package` skill.
