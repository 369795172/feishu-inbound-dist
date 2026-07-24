# feishu-inbound-dist

**Artifacts-only** distribution for the private engine [`369795172/feishu-inbound-skill`](https://github.com/369795172/feishu-inbound-skill).

This repository hosts **GitHub Release wheels** for `feishu-inbound`. Source code is **not** published here.

## Install

Pin a version in your instance requirements, then install the wheel from the matching Release:

```bash
VERSION=0.1.34
pip install "https://github.com/369795172/feishu-inbound-dist/releases/download/v${VERSION}/feishu_inbound-${VERSION}-py3-none-any.whl"
```

Or use the instance helper (public Release first, private engine Release as fallback):

```bash
# ASP
FEISHU_INBOUND_INSTALL=package bash scripts/install_feishu_inbound_engine.sh

# rootgrove personal
FEISHU_INBOUND_INSTALL=package bash tools/feishu_inbound/install_engine.sh
```

## Publish model

| Surface | Repo | Contents |
|---------|------|----------|
| Source (private) | `369795172/feishu-inbound-skill` | Engine source, tests, RFCs |
| Artifacts (public) | `369795172/feishu-inbound-dist` (this repo) | Release wheels only |

Every release is **dual-published**: the same wheel is attached to both the private engine Release and this public dist Release (same tag `vX.Y.Z`).

## Notes

- Wheels are binary distributions; treat them as redistributable build products, not as open source.
- Do not open PRs with source here. Engine changes go to the private skill repo.
