---
name: comfy-cli
description: |
  Generate image and video assets through a local or cloud ComfyUI instance
  using the `comfy` CLI (comfy-cli, installed via `uv tool install comfy-cli`).
  Use when the user wants to run ComfyUI workflows headlessly, script
  generation jobs, download models, or produce image/video assets from a
  saved or exported workflow without opening the ComfyUI web UI. Covers
  workspace routing (local vs cloud), model download, API-format workflow
  authoring, slot overrides, variant fan-out, headless execution, and output
  retrieval. Also covers the common "incomplete metadata, file not fully
  covered" safetensors truncation failure.
  Trigger keywords: comfy, comfyui, comfy cli, comfy workflow, run comfy workflow,
  comfy model download, comfy launch, comfyui asset.
triggers:
  - "comfy cli"
  - "comfyui"
  - "comfy workflow"
  - "run comfy workflow"
  - "comfy model download"
  - "comfyui asset"
  - "comfy launch"
  - "comfy cloud workflow"
od:
  mode: utility
  category: image-generation
  example_prompt: "Run the Z-Image-Turbo text-to-image workflow via comfy-cli to generate a 1024x1024 image and save it to ./outputs."
---

# ComfyUI Asset Generation via `comfy` CLI

Drive a local or cloud ComfyUI instance headlessly with the `comfy` CLI to
produce image and video assets. This skill is a recipe for the agent; it does
not ship models or workflows. Read the reference files before invoking
non-obvious commands.

## Resource map

```text
comfy-cli/
├── SKILL.md
└── references/
    ├── cli-reference.md     # subcommand cheat sheet: run, model, workflow, launch, download
    ├── workflows.md         # obtaining/authoring API-format workflows, slots, set-slot, vary, compose
    └── troubleshooting.md    # safetensors truncation, model-path duplication, MPS, routing
```

Always read `references/cli-reference.md` before running an unfamiliar
subcommand, and `references/troubleshooting.md` when a job fails — the failure
is almost never ComfyUI/PyTorch and almost always a corrupt model file or a
wrong routing target.

## Prerequisites

1. **`comfy` installed.** If `comfy --version` is missing, install it as a uv
   tool (keeps it isolated from any ComfyUI workspace venv):
   ```bash
   uv tool install comfy-cli
   comfy --version
   ```

2. **A routing target.** ComfyUI runs either **local** (a workspace on this
   machine) or **cloud** (Comfy Cloud). Decide once and persist it:
   ```bash
   comfy set-default /path/to/ComfyUI --where local   # local workspace
   # or
   comfy setup --where cloud --api-key <comfy-cloud-key> --non-interactive
   ```
   Per-invocation `--where local|cloud` overrides the persisted default and is
   inherited by subcommands — you do not need to repeat it.

3. **For local: a running server.** `comfy run` needs a reachable ComfyUI
   process. Launch one in the background:
   ```bash
   comfy launch --background
   ```
   `comfy run --host/--port` lets you target an already-running instance
   (e.g. the Comfy Desktop app on 127.0.0.1:8188) without launching your own.

## Workflow

1. **Clarify intent.** What asset (image/video), which workflow, what inputs
   (prompt, dims, seed, model)? If the user only has a UI-format workflow
   exported from the ComfyUI graph editor, that's fine — `comfy run` converts
   UI format to API format client-side.

2. **Ensure models are present and whole.** List, then download any missing
   file by URL. Verify the file is not truncated before trusting it (see
   `references/troubleshooting.md`):
   ```bash
   comfy model list
   comfy model download --url https://huggingface.co/<org>/<repo>/resolve/main/<file>.safetensors \
                        --relative-path models/text_encoders --filename <file>.safetensors
   ```

3. **Obtain an API-format workflow.** Three sources, in order of preference
   (see `references/workflows.md` for detail):
   - **Comfy Cloud saved workflow** → `comfy workflow get <id> --out workflow.json`
   - **Exported from the ComfyUI web UI** (Save → "Export API Format" or the
     frontend-format JSON) → use the file directly; `comfy run` converts it.
   - **Compose from fragments** → `comfy workflow compose blueprint.yaml -o workflow.json`

4. **Inspect and override slots (optional but recommended for agents).**
   A workflow exposes named agent-tweakable slots (prompt text, dims, seed,
   model name, steps). Discover them, then apply overrides without editing
   the JSON by hand:
   ```bash
   comfy workflow slots workflow.json
   comfy workflow set-slot workflow.json 99:62.clip_name="qwen_3_4b.safetensors" \
                                     67.text="a misty harbor at dawn" \
                                     70.seed=724110546647003
   ```

5. **Fan out variants (optional).** To produce N variants in one call (e.g.
   a sweep of seeds or prompts), use `comfy workflow vary` — it emits one
   NDJSON workflow per variant ready to feed to `comfy run`:
   ```bash
   comfy workflow vary workflow.json 70.seed=1,2,3,4 --json-stream
   ```

6. **Validate before submitting (cheap, catches wiring errors).**
   ```bash
   comfy validate --workflow workflow.json
   ```

7. **Run the workflow.** Submit and wait for completion (agents should pass
   `--no-notify`; parse the JSON envelope for the `prompt_id`):
   ```bash
   comfy run --workflow workflow.json --wait --no-notify
   # stdout (with --json) carries the prompt_id; pipe it straight to download:
   comfy run --workflow workflow.json --wait --no-notify --json | jq -r .data.prompt_id | comfy download - --out-dir ./outputs
   ```

8. **Retrieve outputs.** `comfy download <prompt_id> --out-dir ./outputs`
   fetches every output the job produced. Pass `--url-only` if you only need
   URLs to hand to another tool.

9. **Triage failures against `references/troubleshooting.md`** before
   retrying. The two failure modes that account for most agent frustration:
   - `safetensors_rust.SafetensorError: ... incomplete metadata, file not
     fully covered` → a **truncated model download**, not a ComfyUI/MPS bug.
     Verify the file's header-vs-size invariant, delete the bad file, and
     re-fetch with `comfy model download` (or `huggingface-cli`).
   - Job submits but never runs / "connection refused" → wrong `--where`
     target or no local server is running. `comfy launch --background` or
     point `--host/--port` at the running Comfy Desktop instance.

## Output contract

This skill does not emit an OD artifact tag; it produces files on disk under
the user-chosen `--out-dir`. Report back to the user as plain text:
- the command(s) run,
- the `prompt_id`,
- the absolute paths of every downloaded output, and
- any model files fetched (with their final on-disk size).

## Agent guardrails

- Never point `comfy run` at a `*.tmp` / `*.incomplete` / `*.crdownload`
  workflow or model file. If you see one, delete it and re-fetch.
- Never edit an API-format workflow JSON by hand for values that are exposed
  as slots — use `comfy workflow set-slot` so the override is validated.
- Prefer `--json` (or `--json-stream`) for agent-driven calls; the structured
  envelope is parseable and the human-readable side messages go to stderr.
- On Apple Silicon, the MPS backend is fine for execution; do not chase
  MPS/FP8 red herrings when the failure is at `safetensors.safe_open` (that is
  always a file-on-disk problem, before any device is selected).
- Run `comfy model list` and the on-disk header check (in
  `references/troubleshooting.md`) before assuming a model is present and
  whole — partial downloads report normal file names but wrong byte counts.