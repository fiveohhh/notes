# NCS Toolchain: pykwalify Missing on Ubuntu 24.04

## Symptom

`west setup-app` or `west build` fails with an error like:
```
ModuleNotFoundError: No module named 'pykwalify'
```

## Root Cause

The NCS toolchain ships its own Python binary (e.g. `/home/ajlee/ncs/toolchains/b77d8c1312/usr/local/bin/python3`). That binary has Debian's patched `site.py` **frozen into it**. The frozen code only adds `site-packages` to `sys.path` when running inside a virtual environment — otherwise it only looks in `dist-packages`. The toolchain's actual packages (pykwalify, etc.) are installed in `site-packages`, so the toolchain Python can't find them when invoked directly by CMake.

This is Ubuntu 24.04 specific — Ubuntu 22.04 appears to work, likely due to a different frozen bytecode in the toolchain version installed there.

## How Nordic Works Around It

`nrfutil toolchain-manager launch --shell` reads `environment.json` from the toolchain directory and explicitly prepends `site-packages` to `PYTHONPATH`. When using `--shell` everything works; running `west` directly from your own shell skips this step.

## Fix

Before running `west`, export the toolchain's `site-packages` into `PYTHONPATH`:

```bash
export PYTHONPATH=/home/ajlee/ncs/toolchains/b77d8c1312/usr/local/lib/python3.12/site-packages:$PYTHONPATH
```

Or add it to `~/.bashrc` to make it permanent. Adjust the toolchain hash (`b77d8c1312`) and Python version if you upgrade the toolchain.

To find the right path for your installed toolchain:
```bash
ls ~/ncs/toolchains/
# pick the hash, then:
ls ~/ncs/toolchains/<hash>/usr/local/lib/
```
