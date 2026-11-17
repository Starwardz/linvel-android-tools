# Linvel Android Tools

Builds the Node.js and Python toolchains that **run on Android** (aarch64,
bionic libc) for [Coder V2 / Linvel IDE](https://github.com/starwardz) — an
on-device Android IDE that needs standalone `node`/`npm` and `python3`
binaries executable directly on a phone.

The official Linux ARM64 downloads of both are glibc binaries and cannot run
on Android, so both are built from source here with the Android NDK.

## What gets built

| Workflow | Output artifact | Source | Toolchain |
|---|---|---|---|
| `build-node-android.yml` | `node-26.12.1-android-arm64.zip` — `bin/node`, `bin/npm`, `lib/node_modules`, `libc++_shared.so` | Node.js 26.12.1 | NDK r28c, API 24, `--with-intl=small-icu`, `--v8-disable-temporal-support`, 16 KB page alignment |
| `build-python-android.yml` | `python-3.13.7-android-arm64.zip` — install prefix (`bin/`, `lib/libpython*`, stdlib) | CPython 3.13.7 | NDK 28.2.13676358, API 24, CPython's own `Android/android.py` |

Both workflows are manual-only (`workflow_dispatch`): Actions tab → pick a
workflow → Run workflow. They run on public-repo standard runners (4 vCPU,
16 GB RAM, free and unmetered for public repos).

## Checks built into the workflows

- **Node:** the job FAILS unless `out/Release/node` is a bionic binary
  (interpreter `/system/bin/linker64`, no `ld-linux`). A glibc binary would
  be useless on Android, so it never gets packaged.
- **Python:** the log prints a verdict block `STANDALONE PYTHON: YES/NO`.
  Upstream Android support is designed for embedding `libpython`, so whether
  `prefix/bin` holds a usable standalone `python` is an open question this
  build answers. A NO does not fail the job — it means the IDE side needs a
  small launcher binary (same pattern as the JDK `java-launcher`).

## What happens after a green build

1. Download the artifact zip from the Actions run.
2. Upload it to the R2 `coder` bucket as the new tool package
   (`node-26.12.1.zip` / `python-3.13.7.zip`), replacing the dead glibc
   objects currently there.
3. Wire it into Coder V2 (Route C): registry entry, action-native-bin alias,
   packaging entry, precheck entry (`node --version` / `python3 --version`).
   None of that is written until a verified binary exists — writing it
   before the binary is proven would repeat the Node/Rust mistake.

## Known limits

- First builds are experiments: upstream does not test the Android Node path
  in CI, and CPython's Android support is new. Budget time for a failed
  first run and a fix round.
- Standard runners have 14 GB of disk; both workflows free space first.
  Node needs the most (V8 object files).
- No `ccache` yet — a failed late build costs a full rebuild. Add it once
  the first green build exists.
