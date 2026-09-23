# pi-agent image builder

Builds the **pi-agent** container image — the complete Pi
([pi.dev](https://pi.dev/)) harness: the vanilla agent plus baked-in
extensions and runtime mirror configuration — and publishes it to Alibaba
Cloud ACR via the ACR builder service.

Top of the per-image chain split out of the archived
[pi-agent-image](https://github.com/lulin/pi-agent-image) project:

```
deven  ->  pi-vanilla  ->  pi-agent
                        (this repo)
```

## What is in the image

| Layer | Contents |
|-|-|
| Pi extensions | `pi install` of every spec in `PI_PACKAGES` (npm:/git:/URL), stored in the image under `/root/.pi/agent` |
| Patch | runtime mirror configuration written into the image (table below) |
| inherited | vanilla pi from [`pi-vanilla`](../pi-vanilla); toolchain from [`deven`](../deven) |

Patch layer files (all system-wide, applied in the final image — they do not
affect this build):

| Target | File | Values (`*_MIRROR` build args) |
|-|-|-|
| apt | `/etc/apt/sources.list.d/*.sources` | `APT_MIRROR`: ustc (default) / tuna / none |
| npm | global npmrc (`/usr/local/etc/npmrc`) | `NPM_MIRROR`: npmmirror (default) / none — applied *before* the extensions layer so `pi install npm:...` uses it too |
| PyPI (uv) | `/etc/uv/uv.toml` | `PIP_MIRROR`: ustc (default) / tuna / none |
| Cargo | `/opt/cargo/config.toml` | `CARGO_MIRROR`: ustc (default) / tuna / none |

Per-run override without rebuilding, e.g.:
`docker run -e UV_DEFAULT_INDEX=https://pypi.org/simple ...`

## Repository layout

```
Dockerfile    # FROM pi-vanilla (by ACR tag) + extensions + patch layers
              # (from pi-agent-image/aliyuncs/pi-agent)
```

## Building

ACR builder service (云端构建), one rule for this repo:

| Setting | Value |
|-|-|
| Code source | this GitHub repository |
| Dockerfile path | `Dockerfile` |
| Context directory | repo root (no context files are used) |
| Namespace/repo | `***/pi-agent` |
| Tag rules | `latest` (+ version tag matching the pi build) |
| Build args | `PI_PACKAGES="npm:@scope/pkg@1.2.3 git:github.com/u/r@v1"` (space-separated; empty = no baked-in extensions) |

Other optional args: `BASE_IMAGE` (defaults to
`registry.cn-hangzhou.aliyuncs.com/***/pi-vanilla:latest`; same-region
builders can use the `registry-vpc` endpoint) and the four `*_MIRROR` args.

**Ordering:** rebuild `deven` → `pi-vanilla` first when their inputs change —
this image consumes `pi-vanilla:latest` from ACR, it does not rebuild it.

Manual build:

```bash
docker build --build-arg PI_PACKAGES="npm:@foo/bar@1.0.0" -t pi-agent:latest .
```

## Using

```bash
docker pull registry.cn-hangzhou.aliyuncs.com/***/pi-agent:latest
docker run -it --rm -v "$PWD:/workspace" \
  -e ANTHROPIC_API_KEY \
  registry.cn-hangzhou.aliyuncs.com/***/pi-agent:latest
```

Notes:
- Baked-in extensions live in `/root/.pi/agent` **inside the image**; if you
  mount a HOME over it, runtime installs replace the baked set — mount a
  named volume for `/root/.pi` instead if you want persistence.
- Extra pi packages at runtime: `pi install npm:...` (uses the mirrored npm
  registry by default).

## Releasing a new pi version

1. The parent `pi-vanilla` repo bakes and tags the version.
2. Re-trigger this builder; pin `BASE_IMAGE=<...>/pi-vanilla:<x.y.z>` if you
   want an exact rebuild rather than `latest`.
