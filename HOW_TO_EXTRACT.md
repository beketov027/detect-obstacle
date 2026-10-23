# How to restore the original archives

The original large zip files were split into smaller parts (each under 500 MB) in this dataset.

If you extract **every part belonging to one original archive** into the **same output folder**, you get the original folder structure back (for example `cuda_env_py310/...` or `obstacle_detector/...`).

Download the dataset first (example):

```bash
hf download beketov027/detect-obstacle-data --repo-type dataset --local-dir detect-obstacle-data
cd detect-obstacle-data
```

## What you need

- `unzip`
- `zip` (only for multi-volume parts that use `.z01`, `.z02`, …)

On Debian/Ubuntu:

```bash
sudo apt-get install -y unzip zip
```

## Layout

```text
.
├── HOW_TO_EXTRACT.md
├── cuda_env_py310_linux/
├── cuda_env_py310_windows/
├── obstacle_detector_linux/
└── obstacle_detector_windows/
```

Each folder corresponds to one original zip.

There are two kinds of parts:

| Kind | How to recognize | How to extract |
|------|------------------|----------------|
| Self-contained | A single `.zip` with no matching `.z01` | `unzip` directly |
| Multi-volume | `.zip` plus `.z01` / `.z02` / … with the same base name | Reassemble with `zip -s 0`, then `unzip` |

Keep all multi-volume siblings together (same folder, same base name). Do not delete the `.z01` files.

## Multi-volume parts (must reassemble first)

These sets are split across volumes because one file inside was larger than 500 MB:

| Folder | Multi-volume base names |
|--------|-------------------------|
| `cuda_env_py310_linux` | `part01`, `part02`, `part03` |
| `cuda_env_py310_windows` | `part01` (`.z01`–`.z05` + `.zip`) |
| `obstacle_detector_linux` | `part01`, `part02` |
| `obstacle_detector_windows` | `part01`, `part02` |

All other `partXX.zip` files in those folders are self-contained.

## Restore one archive (recommended)

Pick one set, for example Linux obstacle detector:

```bash
SET="obstacle_detector_linux"
SRC="${SET}"                 # folder next to this README
OUT="restored/${SET}"
mkdir -p "${OUT}" "/tmp/zip_reassemble"

# 1) Self-contained parts: unzip straight into OUT
for z in "${SRC}"/${SET}_part*.zip; do
  base="$(basename "${z}" .zip)"
  # Skip if this zip is the last volume of a multi-part set
  if ls "${SRC}/${base}".z[0-9][0-9] >/dev/null 2>&1; then
    continue
  fi
  echo "Extracting ${base}.zip ..."
  unzip -o "${z}" -d "${OUT}"
done

# 2) Multi-volume parts: reassemble, then unzip
for z in "${SRC}"/${SET}_part*.zip; do
  base="$(basename "${z}" .zip)"
  if ! ls "${SRC}/${base}".z[0-9][0-9] >/dev/null 2>&1; then
    continue
  fi
  echo "Reassembling ${base} ..."
  zip -s 0 "${z}" --out "/tmp/zip_reassemble/${base}_full.zip"
  echo "Extracting ${base}_full.zip ..."
  unzip -o "/tmp/zip_reassemble/${base}_full.zip" -d "${OUT}"
  rm -f "/tmp/zip_reassemble/${base}_full.zip"
done

echo "Done. Contents are in: ${OUT}"
```

After this, `${OUT}` matches the original zip’s folder tree.

### Other sets

Change only `SET`:

```bash
SET="cuda_env_py310_linux"
SET="cuda_env_py310_windows"
SET="obstacle_detector_windows"
```

## Manual example (small set)

`cuda_env_py310_windows` has:

- multi-volume: `part01.z01` … `part01.z05` + `part01.zip`
- self-contained: `part02.zip`

```bash
mkdir -p restored/cuda_env_py310_windows

# multi-volume part01
zip -s 0 cuda_env_py310_windows/cuda_env_py310_windows_part01.zip \
  --out /tmp/cuda_env_py310_windows_part01_full.zip
unzip -o /tmp/cuda_env_py310_windows_part01_full.zip \
  -d restored/cuda_env_py310_windows

# self-contained part02
unzip -o cuda_env_py310_windows/cuda_env_py310_windows_part02.zip \
  -d restored/cuda_env_py310_windows
```

## Tips

- Always extract every part of one set into the **same** `-d` directory so paths merge correctly.
- Do not mix Linux and Windows sets into one folder unless you intend to combine those trees.
- `-o` overwrites existing files without prompting (useful when re-running).
- If `unzip` complains about a multi-volume `.zip`, you forgot to run `zip -s 0` first — Info-ZIP often cannot read split volumes directly.
- On Windows, you can use [7-Zip](https://www.7-zip.org/): put all `.z01`/`.z02`/…/`.zip` siblings in one folder, then open the `.zip` and extract. Self-contained `.zip` files extract normally.

## Quick checklist

1. Install `unzip` and `zip`.
2. Choose one of the four split folders.
3. Create one empty output directory.
4. Reassemble every multi-volume group (`zip -s 0 …`), then unzip into the output directory.
5. Unzip every remaining self-contained `.zip` into the same output directory.
6. Confirm the top-level folder (`cuda_env_py310` or `obstacle_detector`) looks complete.
