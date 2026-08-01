# `comfy` CLI reference

Version captured: comfy-cli 1.11.1. Confirm with `comfy --version`. All
subcommands accept `--help` for the full flag list.

## Global flags (apply to every subcommand)

| Flag | Meaning |
|---|---|
| `--workspace <path>` | Path to the ComfyUI workspace to operate on |
| `--recent` / `--here` | Execute from the recent path / current path |
| `--skip-prompt` | Do not prompt the user; use default options (agent-friendly) |
| `--where local\|cloud` | Routing target for this invocation; inherited by subcommands |
| `--json` | Emit a structured JSON envelope on stdout; side messages go to stderr |
| `--json-stream` | One NDJSON event per line; final line is the envelope |
| `--no-json` | Force pretty human output even when auto-detection would pick JSON |
| `--help-json` | Print machine-readable help for the whole CLI and exit |

The persisted defaults are set with `comfy set-default` (workspace path and
`--where`). Per-invocation `--where` overrides them.

## Subcommands at a glance

```
help        Display help for commands
install     Download and install ComfyUI and ComfyUI-Manager
update      Update ComfyUI environment
run         Run an API workflow (submits and returns immediately by default)
validate    Validate an API-format workflow without submitting
upload      Upload files to the ComfyUI server's input directory
download    Download outputs from a completed job (by prompt_id)
run-cli     Walk through the CLI surface with a tiny no-model workflow
stop        Stop background ComfyUI
launch      Launch ComfyUI (--background to daemonize)
setup       Interactive setup wizard — routing, auth, agent skills
set-default Persist workspace path and/or --where
model       Manage models (download / remove / list)
workflow    Slot-based editing of frontend-format workflows
```

## `comfy run` — the main asset-generation entry point

Submits a workflow; returns immediately unless `--wait` is given.

```
--workflow <path>      Required. API-format OR exported UI-format workflow JSON.
                       UI format is converted to API format client-side.
--wait                 Block until the workflow completes.
--notify / --no-notify Desktop notification on completion. Default: on for humans, off for agents.
--verbose              Verbose execution output.
--host <ip>            Target a running ComfyUI (e.g. 127.0.0.1).
--port <int>           Target a running ComfyUI port (e.g. 8188).
--timeout <s>          Per-event silence timeout (default 120). NOT a wall-clock deadline;
                       a workflow that streams progress faster than the timeout runs indefinitely.
--where local|cloud    Routing target.
--api-key <key>        Comfy API key for API/Partner Nodes.
```

Without `--wait`, the command submits and exits; the prompt_id is in the JSON
envelope. With `--wait`, it blocks until completion and the envelope carries
the final status plus the prompt_id for retrieval.

Pipe pattern (agent-friendly):

```bash
comfy run --workflow workflow.json --wait --no-notify --json \
  | jq -r .data.prompt_id \
  | xargs -I{} comfy download {} --out-dir ./outputs
```

## `comfy download <prompt_id>` — retrieve outputs

Reads the prompt_id from the argument or piped stdin.

```
--out-dir <dir>    Where to save outputs. Default ./outputs.
--where local|cloud Routing target.
--url-only          Emit output URLs without downloading files (hand off to other tools).
```

## `comfy model` — model lifecycle

```
comfy model list                         # table of downloaded models
comfy model download --url <url> \
    [--relative-path <dir>]              # path from workspace root
    [--filename <name>]                  # final filename
    [--set-civitai-api-token <token>]    # CivitAI gated models
    [--set-hf-api-token <token>]         # HuggingFace gated models
    [--downloader httpx|aria2]           # httpx default; aria2 needs RPC server
comfy model remove <name>                # or interactive selection
```

`comfy model download` uses httpx and does NOT verify SHA-256. For big
checkpoints where integrity matters, prefer `huggingface-cli download`
(which verifies against HF's manifest) and then point ComfyUI at the file via
`--relative-path`. See `troubleshooting.md` for the header-vs-size check that
confirms any `.safetensors` is whole.

## `comfy workflow` — slot-based editing (agent-friendly)

Operates on **frontend-format** workflow JSON (the ComfyUI graph editor's
native save format). UI-format files passed to `comfy run` are converted to
API format automatically; these commands work on the frontend format.

```
comfy workflow slots <file>                 # list agent-tweakable slots
comfy workflow set-slot <file> ADDR=VALUE... # apply overrides in place (or --stdout)
comfy workflow vary <file> SLOT=v1,v2,...    # N variants as NDJSON
comfy workflow list                          # saved workflows on Comfy Cloud
comfy workflow get <id> --out <path>         # fetch a saved workflow's JSON
comfy workflow save <file>                   # save a local workflow to Comfy Cloud
comfy workflow delete <id>                   # delete a saved workflow from Comfy Cloud
comfy workflow compose <blueprint.yaml>      # compose fragments into one API workflow
comfy workflow decompose <file>              # inverse of compose; emit a reusable fragment
comfy workflow fragment ...                  # inspect/validate fragments
```

Slot addresses look like `99:62.clip_name` or `70.seed` —
`<graphNodeId>.<widgetName>` (or `<subgraphInputName>` for subgraph inputs).
Get the exact addresses from `comfy workflow slots <file>`.

`set-slot` writes back to `<file>` by default; pass `--stdout` to emit the
result instead of mutating the file. `--input <object_info.json>` lets you
introspect slots offline without a running server.

`vary` emits one NDJSON workflow per variant on stdout — pipe each into
`comfy run` for a sweep:

```bash
comfy workflow vary workflow.json 70.seed=1,2,3,4 --json-stream \
  | jq -r .data.workflow_json > variants.ndjson
```

## `comfy launch` / `comfy stop` — server lifecycle

```
comfy launch [--background] [-- <extra ComfyUI args>...]
comfy stop
```

`--background` daemonizes. Extra args after `--` are forwarded to the ComfyUI
process (e.g. `--port 8188`, `--listen 127.0.0.1`,
`--extra-model-paths-config <yaml>`). This is how you target a Comfy Desktop
install or a shared models directory.

## `comfy setup` — one-shot wizard

```
comfy setup --where local|cloud \
            --api-key <comfy-cloud-key> \
            --project-dir <dir> \
            --non-interactive \
            [--skip-skills] [--skip-verify]
```

`--non-interactive` is for CI/agents/scripts. `--skip-skills` skips installing
the comfy agent skill bundle (you usually want to skip this when an agent is
driving).

## `comfy validate` — dry-run a workflow

```
comfy validate --workflow <path>
```

Checks class_types, input shapes, enum values, and edge wiring without
submitting. Cheap; run it before `comfy run` on any untrusted workflow.

## `comfy upload` — put files into the server's input dir

Useful when a workflow consumes an input image (img2img, controlnet, upscale):

```bash
comfy upload --file ./input.png
```

The returned filename is what you wire into the workflow's `LoadImage` node via
`set-slot`.