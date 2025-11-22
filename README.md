Before we start, a note we are looking for open-source contributors to make that happen. We plan to release that cli in December 2025.

# varg cli

> AI video infrastructure from your terminal.

---

## Philosophy

```
simple things should be simple.
complex things should be possible.
everything should be inspectable.
```

---


## Usage

```bash
varg <command> [target] [options]
```

---

## Commands

### `varg run`

Run a model or action.

```bash
# Smart resolve — figures out if it's a model or action
varg run kling --prompt "a cat dancing"
varg run image-to-video --image ./cat.png

# Explicit namespacing (when you need it)
varg run model/kling --prompt "..."
varg run action/image-to-video --image ./cat.png

# Positional args for common patterns
varg run transcribe ./video.mp4
varg run transcribe ./video.mp4 ./output.srt

# Full options
varg run kling \
  --prompt "a cat dancing on the moon" \
  --duration 10 \
  --aspect 16:9 \
  --output ./cat-dance.mp4
```

### `varg ai`

Natural language interface. Let varg figure it out.

```bash
varg ai "animate this cat picture"
varg ai "make a video of a dog surfing, 10 seconds"
varg ai "transcribe my-meeting.mp4 and summarize it"
varg ai "generate 5 variations of this product shot"
```

Under the hood: parses intent → selects model/action → runs it.

### `varg list`

Discover what's available.

```bash
varg list              # everything
varg list models       # only models
varg list actions      # only actions
varg list skills       # only skills
```

**Output:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  varg                                                               │
└─────────────────────────────────────────────────────────────────────┘

  MODELS

  kling             text/image → video         fal · replicate
  flux              text → image               fal
  wan               text/image → video         replicate
  minimax           text → video               fal
  runway            image → video              runway
  elevenlabs        text → voice               elevenlabs
  whisper           audio → text               replicate · fal

  ACTIONS

  image-to-video    animate a still image      kling, wan, runway
  text-to-image     generate an image          flux, sdxl, ideogram
  text-to-video     generate video from text   kling, minimax
  transcribe        speech → text              whisper
  voice             text → speech              elevenlabs
  caption           auto-caption video         whisper + ffmpeg
  upscale           enhance resolution         topaz, real-esrgan

  SKILLS

  product-spin      image → rotating video     flux → kling → ffmpeg
  talking-head      script → avatar video      elevenlabs → hedra
  batch-ads         csv → ad variations        flux → kling (×N)

  ─────────────────────────────────────────────────────────────────────
  12 models · 7 actions · 3 skills · run `varg <cmd> --help` for details
```

### `varg find`

Fuzzy search when you don't know exact names.

```bash
varg find "animate"
varg find "video from image"
varg find "speech"
```

**Output:**

```
┌─ search: "animate" ─────────────────────────────────────────────────┐

  BEST MATCHES

  action/image-to-video     animate a still image
  model/kling               text/image → video (supports animation)
  model/wan                 text/image → video
  model/runway              image → video (motion brush)

  ─────────────────────────────────────────────────────────────────────
  run `varg run <name> --help` for usage
```

### `varg which`

Inspect what's behind an action.

```bash
varg which image-to-video
```

**Output:**

```
┌─ action: image-to-video ────────────────────────────────────────────┐

  Animate a still image with AI.

  ROUTES TO

  kling         default · best quality · 5-10s
  wan           fast · stylized · 5s  
  runway        motion brush · premium

  SELECTION LOGIC

  - default → kling (quality)
  - --fast → wan
  - --provider runway → runway
  - duration > 5s → kling only

  ─────────────────────────────────────────────────────────────────────
  run `varg run image-to-video --help` for full options
```

---

## Inspection

Every runnable has `--help` and `--schema`.

### `--help`

Human-readable documentation.

```bash
varg run kling --help
```

```
┌─ model: kling ──────────────────────────────────────────────────────┐

  Kling 2.5 — video generation by Kuaishou.
  
  USAGE
  
    varg run kling --prompt <text> [options]
    varg run kling --image <path> --prompt <text> [options]

  OPTIONS

    --prompt        what to generate                       required
    --image         input image (enables image-to-video)   optional
    --duration      5 | 10 seconds                         default: 5
    --aspect        16:9 | 9:16 | 1:1                      default: 16:9
    --provider      fal | replicate                        default: fal
    --output        output file path                       default: ./output.mp4

  EXAMPLES

    # Text to video
    varg run kling --prompt "a cat riding a skateboard in tokyo"

    # Image to video  
    varg run kling --image ./cat.png --prompt "cat starts dancing"

    # Full control
    varg run kling \
      --image ./product.png \
      --prompt "product rotates smoothly" \
      --duration 10 \
      --aspect 1:1 \
      --provider replicate \
      --output ./product-spin.mp4
```

### `--schema`

Machine-readable JSON schema. For agents and tooling.

```bash
varg run kling --schema
```

```json
{
  "name": "kling",
  "type": "model",
  "description": "Kling 2.5 — video generation by Kuaishou",
  "input": {
    "type": "object",
    "required": ["prompt"],
    "properties": {
      "prompt": {
        "type": "string",
        "description": "What to generate"
      },
      "image": {
        "type": "string",
        "format": "file-path",
        "description": "Input image for image-to-video mode"
      },
      "duration": {
        "type": "integer",
        "enum": [5, 10],
        "default": 5
      },
      "aspect": {
        "type": "string",
        "enum": ["16:9", "9:16", "1:1"],
        "default": "16:9"
      },
      "provider": {
        "type": "string",
        "enum": ["fal", "replicate"],
        "default": "fal"
      }
    }
  },
  "output": {
    "type": "string",
    "format": "file-path",
    "description": "Path to generated video"
  }
}
```

---

## Resolution

varg resolves names in this order:

```
1. Exact match in models/
2. Exact match in actions/
3. Fuzzy match → suggest
```

Explicit namespacing always works:

```bash
varg run model/kling        # definitely the model
varg run action/transcribe  # definitely the action
```

---

## Config

Optional `varg.config.ts` in project root:

```typescript
export default {
  defaults: {
    provider: 'fal',
    output: './generated',
  },
  models: {
    kling: {
      provider: 'replicate',  // override default provider
      duration: 10,           // override default duration
    }
  },
  aliases: {
    'v': 'image-to-video',
    'tts': 'voice',
  }
}
```

```bash
# With aliases
varg run v --image ./cat.png
varg run tts --text "hello world"
```

---

## Output

Clean, minimal, informative.

**Running:**

```
┌─ kling ─────────────────────────────────────────────────────────────┐
│                                                                     │
│  prompt    "a cat dancing on the moon"                              │
│  duration  10s                                                      │
│  provider  fal                                                      │
│                                                                     │
│  ◐ generating...                                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Complete:**

```
┌─ kling ─────────────────────────────────────────────────────────────┐
│                                                                     │
│  ✓ done in 47s                                                      │
│                                                                     │
│  output  ./cat-moon-dance.mp4                                       │
│  cost    $0.032                                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Error:**

```
┌─ kling ─────────────────────────────────────────────────────────────┐
│                                                                     │
│  ✗ failed                                                           │
│                                                                     │
│  error   content policy violation                                   │
│  prompt  "..." (flagged)                                            │
│                                                                     │
│  try: rephrase prompt or use --provider replicate                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Piping

Unix-friendly. Compose with other tools.

```bash
# Output path to stdout for piping
varg run kling --prompt "cat" --quiet | xargs open

# JSON output for scripting
varg run kling --prompt "cat" --json

# Chain operations
varg run flux --prompt "a cat" --output ./cat.png && \
varg run image-to-video --image ./cat.png --output ./cat.mp4

# Batch from file
cat prompts.txt | xargs -I {} varg run kling --prompt "{}"
```

---

## Environment

```bash
# Required for providers
FAL_KEY=...
REPLICATE_API_TOKEN=...
ELEVENLABS_API_KEY=...

# Optional
VARG_DEFAULT_PROVIDER=fal
VARG_OUTPUT_DIR=./generated
VARG_QUIET=false
```

---

## Installation

```bash
# npm
npm install -g varg

# or run directly
npx varg run kling --prompt "..."

# or in project
bun add varg
bun varg run kling --prompt "..."
```

---

## Skills

Skills are composable workflows — chains of models and actions.

### `varg skills`

```bash
varg skills              # list all skills
varg skills create       # interactive skill builder
varg skills run <name>   # run a skill
varg skills edit <name>  # edit existing skill
```

**Output:**

```
┌─ skills ────────────────────────────────────────────────────────────┐

  product-spin       image → rotating video with shadow
                     flux → kling → ffmpeg

  talking-head       script → avatar video with voice
                     elevenlabs → hedra

  batch-ads          csv → multiple ad variations
                     flux (×N) → kling (×N) → caption

  youtube-short      idea → complete vertical video
                     gpt → flux → kling → caption → music

  ─────────────────────────────────────────────────────────────────────
  4 skills · run `varg skills run <name>` to execute
```

### Skill definition

```yaml
# skills/product-spin.yaml
name: product-spin
description: Create rotating product video with shadow

inputs:
  image:
    type: file
    description: Product image (transparent PNG works best)
  prompt:
    type: string
    default: "product rotates smoothly 360 degrees"

steps:
  - id: enhance
    run: action/upscale
    with:
      image: ${{ inputs.image }}

  - id: generate
    run: model/kling
    with:
      image: ${{ steps.enhance.output }}
      prompt: ${{ inputs.prompt }}
      duration: 5

  - id: loop
    run: ffmpeg
    with:
      input: ${{ steps.generate.output }}
      filter: "loop=loop=3"

output: ${{ steps.loop.output }}
```

### Running skills

```bash
# Basic
varg skills run product-spin --image ./shoe.png

# Override defaults
varg skills run product-spin \
  --image ./shoe.png \
  --prompt "shoe floats and rotates with dramatic lighting"

# Inspect what it will do
varg skills run product-spin --image ./shoe.png --dry-run
```

**Dry run output:**

```
┌─ skill: product-spin (dry-run) ─────────────────────────────────────┐

  STEPS

  1. upscale        action/upscale       ./shoe.png → [upscaled]
  2. generate       model/kling          [upscaled] → [video]
  3. loop           ffmpeg               [video] → [looped]

  ESTIMATED

  time    ~60s
  cost    ~$0.05

  ─────────────────────────────────────────────────────────────────────
  run without --dry-run to execute
```

### Creating skills

```bash
varg skills create
```

Interactive wizard:

```
┌─ new skill ─────────────────────────────────────────────────────────┐

  name: my-workflow
  description: What does this skill do?
  > Creates product videos from images

  ─────────────────────────────────────────────────────────────────────

  Add steps (type 'done' to finish):

  step 1: model/flux
  step 2: model/kling  
  step 3: action/caption
  step 4: done

  ─────────────────────────────────────────────────────────────────────

  ✓ Created skills/my-workflow.yaml
  
  Edit inputs and step config:
    code skills/my-workflow.yaml

```

Or create from natural language:

```bash
varg ai "create a skill that takes a product image, generates 5 angle variations, then creates videos for each"
```

---

## For AI Agents

```typescript
// Get all available tools as JSON schemas
const tools = await $`varg list --json`

// Use in agent
const result = await agent.run({
  tools: JSON.parse(tools),
  prompt: "create a video of a dancing cat"
})
```

```bash
# Schema for any tool
varg run kling --schema > tools/kling.json

# All schemas at once
varg schemas > all-tools.json
```

---

<sub>varg v0.1 · made with ♥ by varg.ai</sub>
