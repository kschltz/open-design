# Workflows for `comfy` — obtaining, authoring, overriding

A **workflow** is the unit of work ComfyUI executes. `comfy run` accepts two
shapes: **API format** (the server's native `{node_id: {class_type, inputs}}`
shape) and **frontend format** (the ComfyUI graph editor's save format, which
carries positions, links, widget values, and subgraph definitions). `comfy run`
converts frontend format to API format client-side, so you can feed it either.

## Three ways to get a workflow

### 1. Pull a saved workflow from Comfy Cloud

```bash
comfy workflow list                          # see your saved workflows
comfy workflow get <id> --out workflow.json  # fetch one to disk
```

Best when the workflow already exists on the cloud and you just want to run it
locally or fan out variants.

### 2. Export from the ComfyUI web UI

In the graph editor: **Save → "Export API Format"** (for `comfy run` directly)
or **Save** (frontend format, for `comfy workflow slots/set-slot/vary`).

Frontend format is the better choice for agents because it carries widget
values and exposes slots. API format is fine if you already have all the
inputs baked in and just want to run it.

The Z-Image-Turbo "Text to Image" template (from the error report) is a
frontend-format workflow with a subgraph whose inputs are `text`, `width`,
`height`, `unet_name`, `clip_name`, `vae_name`, `steps`. Those are the slots you
override.

### 3. Compose from fragments

```bash
comfy workflow compose blueprint.yaml -o workflow.json
comfy workflow decompose workflow.json        # inverse: emit a reusable fragment
comfy workflow fragment ...                   # inspect/validate fragments
```

A blueprint is a YAML file that stitches reusable fragments (loaders, samplers,
decoders) into one workflow. This is the most reproducible path — the
blueprint is diffable and reviewable, where exported JSON is not. Use it when
you want the workflow itself to live in version control.

## Slots — the agent's override surface

A **slot** is a named, agent-tweakable input on a frontend-format workflow.
Addresses look like:

- `99:62.clip_name` — node `62` inside subgraph `99`, widget `clip_name`
- `67.text` — node `67`, widget `text`
- `70.seed` — node `70`, widget `seed`
- `text` — a subgraph input declared on the workflow's `inputs` (no node id)

Discover every slot a workflow exposes:

```bash
comfy workflow slots workflow.json
```

The output lists each slot with its address, type, current value, and (where
known) enum constraints. Read this before guessing addresses.

Apply overrides without hand-editing JSON:

```bash
comfy workflow set-slot workflow.json \
  67.text="a misty harbor at dawn, volumetric light" \
  70.seed=724110546647003 \
  70.steps=8
```

`set-slot` writes back to the file by default. Use `--stdout` to emit the
result instead (non-destructive):

```bash
comfy workflow set-slot workflow.json 67.text="..." --stdout > workflow.run.json
```

`--input <object_info.json>` lets you introspect slots offline (no running
server), which matters in CI and air-gapped setups.

## Variants — fan out in one call

`vary` produces N workflow variants from a per-slot value list, emitted as
NDJSON on stdout:

```bash
comfy workflow vary workflow.json \
  70.seed=1,2,3,4 \
  67.text="harbor at dawn","desert at noon","forest at dusk","city at night" \
  --json-stream > variants.ndjson
```

Each line is a complete workflow JSON. Feed each to `comfy run` for a sweep:

```bash
jq -c .data.workflow_json variants.ndjson | while read wf; do
  echo "$wf" | comfy run --workflow - --wait --no-notify --json
done
```

(`comfy run --workflow -` reads the workflow from stdin when supported; check
`comfy run --help` for the current behavior — otherwise write each variant to
a temp file first.)

## Wiring inputs that are files (img2img, controlnet, upscale)

Workflows that consume an input image need the file in the server's input
directory. Upload first, then set the slot to the returned filename:

```bash
upl=$(comfy upload --file ./input.png --json | jq -r .data.filename)
comfy workflow set-slot workflow.json 71.image="$upl"
```

## Validate before submitting

```bash
comfy validate --workflow workflow.json
```

Catches class_type typos, bad enum values, wrong edge wiring, and shape
mismatches without burning a run. Run it on any workflow you did not author
yourself, and after every `set-slot` sweep that changed node wiring (not just
values).

## When to use which path

| Situation | Use |
|---|---|
| Workflow already on Comfy Cloud | `comfy workflow get` |
| User exported it from the web UI | feed the file to `comfy run` (frontend format preferred so slots work) |
| You want the workflow in version control | `comfy workflow compose` from a YAML blueprint |
| You need a parameter sweep | `comfy workflow vary` then loop `comfy run` |
| You only need one run with tweaked inputs | `comfy workflow set-slot` then `comfy run` |
| Inputs include an image | `comfy upload` then `set-slot` the filename |