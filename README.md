# HostCraft Bootstrap Packages

Custom Termux bootstrap archives repackaged for the HostCraft Android application (`com.hostcraft.android`).

This repository automates the process of downloading official Termux bootstrap packages, patching the default `com.termux` namespace to `com.hostcraft.android` in all text configuration files, and publishing the resulting archives as GitHub Releases.

## How It Works

The build pipeline follows a three-stage process:

1. **Download** -- The official `generate-bootstraps.sh` script from [termux/termux-packages](https://github.com/termux/termux-packages) fetches pre-built `.deb` packages from the Termux APT repository and assembles them into bootstrap ZIP archives for each target architecture.

2. **Patch** -- A Python post-processing script (`patch_prefix.py`) extracts each bootstrap ZIP and replaces all occurrences of `com.termux` with `com.hostcraft.android` in text files (shell scripts, config files, symlink maps, dpkg metadata, etc.). ELF binaries are detected via magic byte inspection and left untouched.

3. **Integrate frpc** -- The pipeline queries `https://api.github.com/repos/fatedier/frp/releases/latest`, downloads the matching `frpc` executable for each arch, and installs it as `bin/frpc` (mode `0755`) with a version stamp at `bin/FRPC_VERSION`. Mapping (first hit wins):

   - `aarch64` <- `android_arm64` (fallback `linux_arm64`)
   - `arm` <- `linux_arm` (fallback `linux_arm_hf`, no 32-bit android asset published)
   - `x86_64` <- `linux_amd64` (emulators, no android asset published)

   Pin a specific version via manual trigger input `frpc_version` (e.g. `v0.71.0`); default is `latest`.

4. **Release** -- The patched archives are re-zipped and published as GitHub Release assets, ready to be downloaded by the HostCraft app at install time.

## Architectures

Bootstrap archives are built for the following Android CPU architectures:

| Architecture | Description             |
|--------------|-------------------------|
| `aarch64`    | 64-bit ARM (most devices) |
| `arm`        | 32-bit ARM              |
| `x86_64`     | 64-bit x86 (emulators)  |

## Build Schedule

The pipeline runs automatically:

- On every push to `main`
- On a recurring schedule (every 5 days)
- On manual trigger via `workflow_dispatch`

Manual triggers allow overriding the target package name and pinning the frpc version (`frpc_version`, default `latest`) if needed.

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── build_bootstrap.yml   # GitHub Actions pipeline
├── patch_prefix.py               # Post-processing script
├── LICENSE
└── README.md
```

## What Gets Patched

The `patch_prefix.py` script modifies only **text files** inside the bootstrap. Examples of patched content:

- APT repository configuration (`etc/apt/sources.list`)
- Shell profile scripts (`etc/profile`, `etc/bash.bashrc`)
- dpkg status and package metadata (`var/lib/dpkg/status`, `var/lib/dpkg/info/*.list`)
- Symlink map (`SYMLINKS.txt`)
- Termux utility scripts (`bin/termux-*`)

**ELF binaries are not modified.** Compiled programs like `bash`, `apt`, `dpkg`, and shared libraries retain their original hardcoded paths (`/data/data/com.termux/files/usr`). The HostCraft app handles this at runtime by setting `LD_LIBRARY_PATH`, `PATH`, and running bash in `--posix` mode with a custom `ENV` variable.

## Using the Bootstrap Archives

Download the latest release asset for your target architecture:

```
https://github.com/<owner>/hostcraft-bootstrap-packages/releases/latest/download/bootstrap-aarch64.zip
```

The ZIP contains the full `usr/` prefix tree plus the bundled `bin/frpc` client (see `bin/FRPC_VERSION` for the exact upstream tag). Extract it to your app's files directory:

```
/data/data/com.hostcraft.android/files/usr/
```

## License

This project is licensed under the Apache License 2.0. See [LICENSE](LICENSE) for details.

The bootstrap packages themselves are built from [termux-packages](https://github.com/termux/termux-packages) and are subject to their respective upstream licenses.
