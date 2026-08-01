# ComfyUI troubleshooting for agents

Most ComfyUI failures an agent will hit are **file-on-disk problems**, not
framework/PyTorch/device problems. Triage these first before chasing
red herrings.

## 1. `safetensors_rust.SafetensorError: incomplete metadata, file not fully covered`

**What it means.** A `.safetensors` file is `[8-byte LE u64 header_len][JSON
header][raw tensor bytes]`. The header maps each tensor name to
`{dtype, shape, data_offsets: [start, end]}`. The loader enforces one
invariant:

```
8 + header_len + max(data_offsets[1])  ==  os.path.getsize(file)
```

When that fails, `safe_open` refuses to mmap the file. The file on disk is
structurally inconsistent. This is raised at load time, **before** any device
is selected — so it is never an MPS/CUDA/FP8 problem.

**Causes in order of likelihood:**

1. **Truncated / interrupted download.** The model manager (Comfy Desktop, HF
   Hub, `comfy model download` with httpx) left a partial file when the
   network dropped, the app was quit, or the machine slept mid-download.
2. **Appended / overwritten file** whose size exceeds the header's declared
   extent (a second incomplete download wrote past the original EOF; header
   is stale).
3. **Pointing at a stale / wrong file** — a `*.tmp`, `*.crdownload`,
   `*.incomplete`, or a copy from a different repo with a different header.
   Especially likely when there are **two model trees** (a shared
   `ComfyUI-Shared/models` and an install-local `ComfyUI/models`); the loader
   may resolve the bad copy.
4. Genuine disk corruption (rare; treat as #1 — re-download fixes it).

**Signal:** when the same error hits *different* files at different call
sites (CLIPLoader then UNetLoader), it is one root cause — an interrupted
**pack** download — not two separate problems. Re-fetch the whole pack, not
just the first file in the traceback.

**Confirm (stdlib only, no `safetensors` import needed):**

```python
import os, struct, json
def check(path):
    size = os.path.getsize(path)
    with open(path, "rb") as f:
        (hlen,) = struct.unpack("<Q", f.read(8))
        hdr = json.loads(f.read(hlen))
    declared_end = max((m["data_offsets"][1] for m in hdr.values()
                        if "data_offsets" in m), default=0)
    expected = 8 + hlen + declared_end
    verdict = ("OK" if size == expected else
              f"TRUNCATED by {expected-size:,}" if size < expected else
              f"OVERSIZED by {size-expected:,}")
    print(f"{os.path.basename(path)}: {size:,} vs {expected:,} -> {verdict}")
for p in ["/path/to/text_encoders/qwen_3_4b.safetensors",
          "/path/to/diffusion_models/z_image_turbo_bf16.safetensors",
          "/path/to/vae/ae.safetensors"]:
    if os.path.exists(p): check(p)
```

Compare on-disk size to the HuggingFace *Files and versions* tab — anything
materially under the listed size is truncated.

**Fix:**

```bash
# delete the corrupt files AND any half-written temp siblings, across BOTH trees
find ~/ComfyUI-Shared/models /path/to/ComfyUI/models \
  -type f \( -name "<bad>.safetensors" \
             -o -name "*.incomplete" -o -name "*.tmp" \
             -o -name "*.crdownload" -o -name "*.partial" \) -print -delete

# re-fetch with integrity verification (preferred over httpx-only download)
huggingface-cli download Comfy-Org/z_image_turbo \
  --include "split_files/text_encoders/qwen_3_4b.safetensors" \
  --local-dir ./z_image_pack
# or, for non-HF sources:
comfy model download --url <url> --relative-path models/<dir> --filename <name>.safetensors
```

`huggingface-cli` verifies SHA-256 against HF's manifest; `comfy model
download` (httpx) does not. Always run the header check after a `comfy model
download` of a large file.

## 2. Job submits but never runs / "connection refused"

Wrong routing target, or no local server is running.

- `comfy --where local` but no ComfyUI process is up → `comfy launch --background`
- Want to target the Comfy Desktop app instead → `comfy run --host 127.0.0.1 --port 8188`
- `comfy --where cloud` but no API key → `comfy setup --where cloud --api-key <key> --non-interactive`

Check the persisted default: `comfy set-default` shows the workspace path and
`--where`. A per-invocation `--where` overrides it for that command and its
subcommands.

## 3. "Model not found" in the picker, but the file is on disk

ComfyUI scans specific subdirectories under the workspace's `models/`:
`checkpoints`, `text_encoders`, `diffusion_models`, `vae`, `loras`, `clip`,
`clip_vision`, `controlnet`, `embeddings`, `unet`, `upscale_models`, … A file
in the wrong subdirectory is invisible to the loader that expects it there.

- `comfy model list` shows what ComfyUI actually sees.
- The Z-Image-Turbo pack layout is:
  - `models/text_encoders/qwen_3_4b.safetensors`
  - `models/diffusion_models/z_image_turbo_bf16.safetensors`
  - `models/vae/ae.safetensors`

If you have two model trees, point ComfyUI at the canonical one via
`--extra-model-paths-config <yaml>` (passed through `comfy launch -- --extra-model-paths-config ...`)
and remove the duplicates from the install-local tree to avoid the
"loaded the wrong copy" failure mode.

## 4. Apple Silicon (MPS) red herrings

On macOS, ComfyUI runs on the MPS backend. These are **not** problems to chase
when a failure is at `safetensors.safe_open`:

- `comfy-aimdo unsupported operating system: Darwin` — informational; the
  AIMDO fast paths are CUDA-only and fall back silently.
- `Could not autodetect AIMDO implementation, assuming Nvidia` — same; informational.
- `comfy_kitchen backend cuda: ... Extension file not found: .../cuda/_C.abi3.so` —
  expected on macOS; the `eager` backend is the one in use.
- `comfy_kitchen backend triton: ... ImportError: No module named 'triton'` —
  expected; Triton is CUDA-only.

The one MPS-specific runtime crash worth knowing is the **Float8 dequantize
on MPS** crash (`Trying to convert Float8_e4m3fn to the MPS backend`) when a
quantized model is loaded. That is a runtime crash inside a node, not a
load-time `safe_open` error, and it has its own patch (see the
`fix_comfy_fp8_mps` tool / `comfy_kitchen` MPS FP8 dequantize patch). Do not
confuse it with the truncation error above — they happen at different stages.

## 5. `comfy run` hangs / times out

`--timeout <s>` is a **per-event silence** timeout, not a wall-clock deadline.
A workflow that streams progress events faster than the timeout runs
indefinitely. If a real run hangs:

- Check `comfy launch` is up and the port matches (`comfy run --host/--port`).
- Run `comfy validate --workflow <path>` first to rule out a malformed graph.
- Look at the ComfyUI server log (stdout of `comfy launch`) for the actual
  error — `comfy run`'s client-side envelope may only report silence.
- For very large model loads on MPS, the first run can take minutes of
  apparent silence before the first event; raise `--timeout` rather than
  assuming a hang.

## 6. Slot override silently does nothing

- You ran `set-slot` on an **API-format** workflow. Slots are a
  **frontend-format** feature. Re-export from the UI in frontend format, or
  use `comfy workflow decompose`/`compose` to work in frontend format.
- The address is wrong. Always run `comfy workflow slots <file>` first and
  copy the exact address — subgraph nodes are `99:62.widget`, not `62.widget`.
- The widget name doesn't match the current ComfyUI version (node authors
  rename widgets). `slots` shows the live names; trust it over old docs.

## 7. `comfy model download` succeeds but the file is still bad

httpx downloads do not verify checksums. A network blip mid-download can
produce a file that looks complete (right size, valid 8-byte header length)
but is internally truncated. Always run the §1 header check after any
`comfy model download` of a file >1 GB. For integrity, prefer
`huggingface-cli download` (SHA-256 verified) when the source is HuggingFace.