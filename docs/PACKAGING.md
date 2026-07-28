# Packaging — dual GitHub Release wheels (private + public)

GitHub **does not** host Python packages on GitHub Packages. Engine releases are published as **wheels attached to GitHub Releases** on two surfaces:

| Surface | Repo | Visibility | Role |
|---------|------|------------|------|
| Source / private Release | `369795172/feishu-inbound-skill` | private | Engine SSOT (source, tests) + backup wheel |
| **Public dist Release** | `369795172/feishu-inbound-dist` | **public** | **Wheels + documentation** (this repo) |

Instance repos **only pin a version** in `requirements-feishu-inbound.txt`, e.g. `feishu-inbound==0.1.37`. Install helpers prefer the **public** wheel URL, then fall back to the private engine Release.

**Consumer docs** (no source access): [INDEX.md](./INDEX.md) · [GETTING_STARTED.md](./GETTING_STARTED.md) · [skills/](../skills/)

## Publish (maintainer)

Skill (meta): rootgrove `rules/skills/workflow_feishu_inbound_engine_release.md`  
Gate script: `./scripts/release_engine.sh` (from this repo).

1. Detect drift: `python3 scripts/prepare_engine_release.py --check` (bump with `--prepare` if needed).
2. Bump `version` in `pyproject.toml` and `src/feishu_inbound/__init__.py` (or let `--prepare` do it).
3. Update `docs/CHANGELOG.md`, commit to `main`.
4. Run `./scripts/release_engine.sh --dry-run` then `./scripts/release_engine.sh`.

When `main` already has a higher `pyproject` version than the latest Release, push to `main` also triggers **Auto-tag when version ahead of release**, which creates `vX.Y.Z` and lets **Publish Python Package** attach the wheel to the private Release and mirror it to the public dist (when `FEISHU_INBOUND_DIST_TOKEN` is set).

`release_engine.sh` also verifies the public dist asset and, if missing, runs `scripts/mirror_wheel_to_dist.sh`.

### GHA secret for public mirror

On `369795172/feishu-inbound-skill`, set repository secret **`FEISHU_INBOUND_DIST_TOKEN`**:

- Fine-grained PAT (recommended): Contents **Read and write** on `369795172/feishu-inbound-dist` only
- Or classic PAT with `repo` (broader; avoid if possible)

Without this secret, Publish workflow still uploads the private Release and emits a warning; maintainers can mirror with:

```bash
./scripts/mirror_wheel_to_dist.sh 0.1.34
```

## Install (instance / local)

**Preferred (no token):** public dist bare URL.

```bash
VERSION=0.1.34
pip install "https://github.com/369795172/feishu-inbound-dist/releases/download/v${VERSION}/feishu_inbound-${VERSION}-py3-none-any.whl"
```

**ASP instance helper** (reads version from `requirements-feishu-inbound.txt`; public first, private fallback):

```bash
cd projects/asp-infra
source scripts/load_asp_env.sh   # optional: other secrets
FEISHU_INBOUND_INSTALL=package bash scripts/install_feishu_inbound_engine.sh
```

Set `FEISHU_INBOUND_INSTALL=package` to force release-wheel install even when a local `projects/feishu-inbound-skill` checkout exists.

**Private fallback auth** (only if public dist miss): PAT with **`repo`** read on `369795172/feishu-inbound-skill`. `read:packages` is **not** required.

```bash
export FEISHU_INBOUND_GH_TOKEN="<pat-with-repo-read>"
# or: export FEISHU_INBOUND_GH_TOKEN="$(gh auth token)"
```

## GitHub Actions (AI-MYG/asp)

Public dist means most CI installs need **no** engine-repo token. Keep `FEISHU_INBOUND_GH_TOKEN` only if you want private fallback. Example:

```yaml
- name: Install feishu-inbound engine
  env:
    FEISHU_INBOUND_INSTALL: package
    # optional private fallback:
    # FEISHU_INBOUND_GH_TOKEN: ${{ secrets.FEISHU_INBOUND_GH_TOKEN }}
  run: bash scripts/install_feishu_inbound_engine.sh
```

## Local editable dev (unchanged)

```bash
pip install -e projects/feishu-inbound-skill
```
