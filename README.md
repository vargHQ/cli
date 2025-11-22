# varg cli

> AI video infrastructure from your terminal.

Status

🟡 Pre-release — Active development, looking for contributors.


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

## Commands Overview

| Command | Description |
|---------|-------------|
| `varg run` | Run a model, action, or skill |
| `varg jobs` | Manage running and completed jobs |
| `varg ai` | Natural language interface |
| `varg list` | Discover available tools |
| `varg find` | Fuzzy search |
| `varg which` | Inspect action routing |
| `varg skills` | Manage workflows |
| `varg schemas` | Export all schemas |

---

## `varg run`

Run a model or action.

```bash
# Smart resolve — figures out if it's a model or action
varg run kling --prompt "a cat dancing"
varg run image-to-video --image ./cat.png

# Explicit namespacing
varg run model/kling --prompt "..."
varg run action/image-to-video --image ./cat.png
varg run skill/product-spin --image ./shoe.png

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

### Execution Modes

```bash
# SYNC (default for fast tasks < 30s)
varg run flux --prompt "a cat"
# → waits, shows progress, returns result

# ASYNC (default for slow tasks > 30s)  
varg run kling --prompt "a cat"
# → returns job id immediately

# Force behaviors
varg run kling --prompt "cat" --wait      # force sync
varg run flux --prompt "cat" --async      # force async
varg run kling --prompt "cat" --watch     # async + follow progress
```

### Execution Options

```bash
--wait          # block until complete
--async         # return job id immediately  
--watch         # follow progress in real-time
--quiet         # minimal output, just result path
--json          # output as JSON
--output <path> # custom output path
--dry-run       # show what would happen
```

---

## `varg jobs`

Manage running and completed jobs.

```bash
varg jobs                      # list all jobs
varg jobs <id>                 # status of specific job
varg jobs <id> --watch         # attach and follow progress
varg jobs <id> --logs          # show full logs
varg jobs <id> --output        # print output path
varg jobs <id> --kill          # cancel job
varg jobs <id> --retry         # retry failed job
varg jobs clean                # remove completed jobs
varg jobs clean --all          # remove everything
varg jobs clean --before 7d    # remove older than 7 days
```

### List Jobs

```bash
varg jobs
```

```
┌─ jobs ──────────────────────────────────────────────────────────────┐

  RUNNING

  abc123    kling           ◐ generating         1:34 / ~2:00
  def456    skill/batch     ◐ step 2/4           flux (3/10)

  QUEUED

  ghi789    image-to-video  ◷ waiting            depends: abc123

  RECENT

  jkl012    flux            ✓ done      2m ago   ./out/cat.png
  mno345    transcribe      ✓ done      5m ago   ./out/subs.srt
  pqr678    kling           ✗ failed    8m ago   content policy

  ─────────────────────────────────────────────────────────────────────
  2 running · 1 queued · 3 recent

  varg jobs <id> for details · varg jobs clean to clear
```

### Job Status

```bash
varg jobs abc123
```

```
┌─ job: abc123 ───────────────────────────────────────────────────────┐

  TYPE      model/kling
  PROMPT    "a cat dancing on the moon"
  PROVIDER  fal

  TIMELINE

  ✓ created                                       10:32:01
  ✓ queued                                        10:32:01
  ✓ started                                       10:32:04
  ✓ preprocessing                                 10:32:08
  ◐ generating                                    10:32:15    now
  ◯ postprocessing
  ◯ complete

  PROGRESS

  ████████████████░░░░░░░░░░░░░░  47/120 frames   39%

  TIME

  elapsed     1:34
  remaining   ~2:26

  COST        ~$0.032

  ─────────────────────────────────────────────────────────────────────
  --watch to follow · --logs for details · --kill to cancel
```

### Watch Mode

```bash
varg jobs abc123 --watch
# or
varg run kling --prompt "cat" --watch
```

```
┌─ abc123 · kling ────────────────────────────────────────────────────┐
│                                                                     │
│  "a cat dancing on the moon"                                        │
│                                                                     │
│  ✓ queued                                            0:00           │
│  ✓ started                                           0:03           │
│  ✓ preprocessing                                     0:05           │
│  ◐ generating                                        1:34           │
│  ◯ postprocessing                                                   │
│  ◯ complete                                                         │
│                                                                     │
│  ████████████████░░░░░░░░░░░░░░  47/120    39%    eta 2:26          │
│                                                                     │
│  cost ~$0.032                                                       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
  ctrl+c detach (job continues) · ctrl+x kill job
```

### Completed Job

```
┌─ abc123 · kling ────────────────────────────────────────────────────┐
│                                                                     │
│  ✓ done                                                             │
│                                                                     │
│  output   ./generated/abc123-cat-moon.mp4                           │
│  time     4:02                                                      │
│  cost     $0.032                                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Failed Job

```
┌─ pqr678 · kling ────────────────────────────────────────────────────┐
│                                                                     │
│  ✗ failed at: generating                                            │
│                                                                     │
│  error     content policy violation                                 │
│  prompt    "..." (flagged)                                          │
│  provider  fal                                                      │
│                                                                     │
│  try:                                                               │
│    · rephrase prompt                                                │
│    · --provider replicate (different policy)                        │
│    · --retry to try again                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Job Chaining

### `--then` flag

```bash
# Run command when job completes
varg run flux --prompt "a cat" \
  --then "varg run kling --image \$OUTPUT"

# Multiple chains
varg run flux --prompt "a cat" \
  --then "varg run kling --image \$OUTPUT" \
  --then "varg run caption --video \$OUTPUT"
```

### Job References

```bash
# Start first job
varg run flux --prompt "a cat"
# → job:abc123

# Reference previous job output
varg run kling --image job:abc123
# → job:def456 (waits for abc123)
```

```
┌─ def456 · kling ────────────────────────────────────────────────────┐
│                                                                     │
│  ◷ waiting for dependency                                           │
│                                                                     │
│  DEPENDS ON                                                         │
│                                                                     │
│  abc123    flux    ◐ 67%    eta 0:12                                │
│                                                                     │
│  ─────────────────────────────────────────────────────────────────  │
│  starts automatically when abc123 completes                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Parallel Jobs

```bash
# Start multiple jobs
varg run flux --prompt "cat 1" --async   # → job:a
varg run flux --prompt "cat 2" --async   # → job:b  
varg run flux --prompt "cat 3" --async   # → job:c

# Wait for all
varg jobs a b c --wait

# Or watch all
varg jobs a b c --watch
```

```
┌─ watching 3 jobs ───────────────────────────────────────────────────┐
│                                                                     │
│  a    flux    ████████████████████░░░░░░░░░░  67%    eta 0:08       │
│  b    flux    ██████████░░░░░░░░░░░░░░░░░░░░  33%    eta 0:16       │
│  c    flux    ████████████████████████████░░  93%    eta 0:02       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Callbacks

### Webhooks

```bash
varg run kling --prompt "cat" \
  --webhook https://api.myapp.com/varg/done
```

Webhook payload:

```json
{
  "job_id": "abc123",
  "status": "completed",
  "output": "https://storage.varg.ai/abc123/output.mp4",
  "output_local": "./generated/abc123.mp4",
  "elapsed": 127,
  "cost": 0.032
}
```

### Shell Commands

```bash
# On success
varg run kling --prompt "cat" \
  --on-done "notify-send 'Video ready: \$OUTPUT'"

# On failure  
varg run kling --prompt "cat" \
  --on-fail "echo 'Job \$JOB_ID failed: \$ERROR' >> errors.log"

# Both
varg run kling --prompt "cat" \
  --on-done "open \$OUTPUT" \
  --on-fail "say 'generation failed'"
```

Available variables: `$JOB_ID`, `$OUTPUT`, `$STATUS`, `$ERROR`, `$COST`, `$ELAPSED`

---

## `varg ai`

Natural language interface.

```bash
varg ai "animate this cat picture"
varg ai "make a 10 second video of a dog surfing"
varg ai "transcribe meeting.mp4 and summarize it"
varg ai "generate 5 product shot variations"
varg ai "create a skill for batch video generation"
```

```
┌─ varg ai ───────────────────────────────────────────────────────────┐
│                                                                     │
│  "animate this cat.png"                                             │
│                                                                     │
│  PLAN                                                               │
│                                                                     │
│  1. image-to-video                                                  │
│     model: kling                                                    │
│     image: ./cat.png                                                │
│     prompt: "subtle natural movement"                               │
│                                                                     │
│  proceed? [Y/n]                                                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

```bash
# Skip confirmation
varg ai "animate cat.png" --yes

# Just show plan, don't execute
varg ai "animate cat.png" --plan
```

---

## `varg list`

Discover available tools.

```bash
varg list              # everything
varg list models       # only models
varg list actions      # only actions
varg list skills       # only skills
varg list --json       # machine-readable
```

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
  12 models · 7 actions · 3 skills
  
  varg run <name> --help for usage
```

---

## `varg find`

Fuzzy search.

```bash
varg find "animate"
varg find "video from image"
varg find "speech"
```

```
┌─ search: "animate" ─────────────────────────────────────────────────┐

  BEST MATCHES

  action/image-to-video     animate a still image
  model/kling               text/image → video
  model/wan                 text/image → video
  model/runway              image → video (motion brush)

  ─────────────────────────────────────────────────────────────────────
  varg run <name> --help for usage
```

---

## `varg which`

Inspect action routing.

```bash
varg which image-to-video
```

```
┌─ action: image-to-video ────────────────────────────────────────────┐

  Animate a still image with AI.

  ROUTES TO

  kling         default · best quality · 5-10s
  wan           fast · stylized · 5s
  runway        motion brush · premium

  SELECTION LOGIC

  default        → kling
  --fast         → wan
  --model runway → runway
  duration > 5s  → kling only

  ─────────────────────────────────────────────────────────────────────
  varg run image-to-video --help for full options
```

---

## `varg skills`

Composable workflows.

```bash
varg skills                    # list skills
varg skills run <name>         # run a skill
varg skills create             # interactive builder
varg skills show <name>        # show definition
varg skills edit <name>        # edit skill
```

### List Skills

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
  4 skills · varg skills run <name> to execute
```

### Run Skill

```bash
# Basic
varg skills run product-spin --image ./shoe.png

# Override inputs
varg skills run product-spin \
  --image ./shoe.png \
  --prompt "dramatic lighting, slow rotation"

# Preview
varg skills run product-spin --image ./shoe.png --dry-run

# Watch execution
varg skills run product-spin --image ./shoe.png --watch
```

### Dry Run

```
┌─ skill: product-spin (dry-run) ─────────────────────────────────────┐

  INPUT   ./shoe.png

  STEPS

  1. enhance      action/upscale      ./shoe.png → [upscaled]
  2. generate     model/kling         [upscaled] → [video]
  3. loop         ffmpeg              [video] → [looped]

  ESTIMATE

  time    ~90s
  cost    ~$0.05

  ─────────────────────────────────────────────────────────────────────
  run without --dry-run to execute
```

### Watch Skill

```
┌─ skill: product-spin ───────────────────────────────────────────────┐
│                                                                     │
│  INPUT    ./shoe.png                                                │
│                                                                     │
│  STEPS                                                              │
│                                                                     │
│  ✓ enhance     upscale     done       8s      ./tmp/enh.png         │
│  ◐ generate    kling       47/120     1:34                          │
│  ◯ loop        ffmpeg      waiting                                  │
│                                                                     │
│  ████████████████░░░░░░░░░░░░░░  step 2/3   eta ~1:20               │
│                                                                     │
│  cost ~$0.04                                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
  ctrl+c detach · ctrl+x cancel
```

### Skill Definition

```yaml
# skills/product-spin.yaml
name: product-spin
description: Create rotating product video with shadow

inputs:
  image:
    type: file
    required: true
    description: Product image (transparent PNG best)
  prompt:
    type: string
    default: "product rotates smoothly 360 degrees"
  duration:
    type: number
    default: 5

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
      duration: ${{ inputs.duration }}

  - id: loop
    run: ffmpeg
    with:
      input: ${{ steps.generate.output }}
      filter: "loop=loop=3"

output: ${{ steps.loop.output }}
```

### Parallel Steps

```yaml
# skills/batch-thumbnails.yaml
name: batch-thumbnails

inputs:
  prompts:
    type: array

steps:
  - id: generate
    run: model/flux
    foreach: ${{ inputs.prompts }}    # loop over array
    parallel: true                     # run in parallel
    with:
      prompt: ${{ item }}

  - id: zip
    run: action/zip
    with:
      files: ${{ steps.generate.outputs }}

output: ${{ steps.zip.output }}
```

### Create Skill

```bash
varg skills create
```

```
┌─ new skill ─────────────────────────────────────────────────────────┐

  name: my-workflow
  description: What does this skill do?
  > Batch generate product videos

  ─────────────────────────────────────────────────────────────────────

  steps (type 'done' when finished):

  1: model/flux
  2: model/kling
  3: action/caption
  4: done

  ─────────────────────────────────────────────────────────────────────

  ✓ created skills/my-workflow.yaml

  edit: code skills/my-workflow.yaml
```

Or via natural language:

```bash
varg ai "create a skill that takes product images and generates rotating videos with captions"
```

---

## Inspection

Every runnable has `--help` and `--schema`.

### `--help`

```bash
varg run kling --help
```

```
┌─ model: kling ──────────────────────────────────────────────────────┐

  Kling 2.5 — video generation by Kuaishou

  USAGE

    varg run kling --prompt <text> [options]
    varg run kling --image <path> --prompt <text> [options]

  INPUT OPTIONS

    --prompt        what to generate                       required
    --image         input image (enables i2v mode)         optional
    --duration      5 | 10 seconds                         default: 5
    --aspect        16:9 | 9:16 | 1:1                      default: 16:9
    --provider      fal | replicate                        default: fal

  OUTPUT OPTIONS

    --output        output file path                       auto
    --quiet         only print result path
    --json          output as JSON

  EXECUTION OPTIONS

    --wait          wait for completion
    --async         return job id immediately
    --watch         follow progress live
    --then <cmd>    run command on completion
    --webhook <url> POST to URL on completion

  EXAMPLES

    # text to video
    varg run kling --prompt "a cat skateboarding in tokyo"

    # image to video
    varg run kling --image ./cat.png --prompt "cat starts dancing"

    # full control
    varg run kling \
      --image ./product.png \
      --prompt "product rotates smoothly" \
      --duration 10 \
      --provider replicate \
      --output ./spin.mp4 \
      --watch
```

### `--schema`

```bash
varg run kling --schema
```

```json
{
  "name": "kling",
  "type": "model",
  "description": "Kling 2.5 — video generation by Kuaishou",
  "async": true,
  "estimatedTime": "60-180s",
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
    "type": "object",
    "properties": {
      "file": {
        "type": "string",
        "format": "file-path"
      },
      "url": {
        "type": "string",
        "format": "uri"
      }
    }
  }
}
```

### `varg schemas`

Export all schemas at once.

```bash
varg schemas                   # all schemas
varg schemas models            # only models
varg schemas actions           # only actions
varg schemas > tools.json      # save to file
```

---

## Piping & Scripting

Unix-friendly.

```bash
# Quiet mode — just output path
varg run flux --prompt "cat" --quiet --wait
# → ./generated/abc123.png

# JSON for scripting
varg run flux --prompt "cat" --json --wait | jq '.output'

# Pipe to other tools
varg run flux --prompt "cat" --quiet --wait | xargs open

# Chain with &&
varg run flux --prompt "cat" -o ./cat.png --wait && \
varg run kling --image ./cat.png --watch

# Batch from file
cat prompts.txt | while read p; do
  varg run flux --prompt "$p" --async
done

# Process job outputs
varg jobs --json | jq '.[] | select(.status == "completed") | .output'
```

---

## Resolution

Name resolution order:

```
1. Exact match in models/
2. Exact match in actions/
3. Exact match in skills/
4. Fuzzy match → suggest
```

Explicit namespacing:

```bash
varg run model/kling
varg run action/transcribe
varg run skill/product-spin
```

---

## Config

`varg.config.ts` in project root:

```typescript
import { defineConfig } from 'varg'

export default defineConfig({
  defaults: {
    provider: 'fal',
    output: './generated',
  },

  models: {
    kling: {
      provider: 'replicate',
      duration: 10,
    }
  },

  jobs: {
    retention: '7d',
    maxConcurrent: 5,
    defaultMode: 'async',
  },

  aliases: {
    'v': 'image-to-video',
    'i2v': 'image-to-video',
    'tts': 'voice',
  },

  hooks: {
    onComplete: 'notify-send "varg: $JOB_ID done"',
    onFail: 'echo "$JOB_ID: $ERROR" >> ~/.varg/errors.log',
  }
})
```

```bash
# With aliases
varg run v --image ./cat.png
varg run tts --text "hello world"
```

---

## Environment

```bash
# Provider keys
FAL_KEY=...
REPLICATE_API_TOKEN=...
ELEVENLABS_API_KEY=...
RUNWAY_API_KEY=...

# Optional
VARG_DEFAULT_PROVIDER=fal
VARG_OUTPUT_DIR=./generated
VARG_ASYNC=true
VARG_QUIET=false
```

---

## Installation

```bash
# npm
npm install -g varg

# bun
bun add -g varg

# npx (no install)
npx varg run flux --prompt "cat"

# homebrew (coming soon)
brew install varg
```

---

## For AI Agents

```typescript
// Get all tools as JSON schemas
const tools = await $`varg schemas --json`

// Run with agent
const result = await agent.run({
  tools: JSON.parse(tools),
  prompt: "create a video of a dancing cat"
})
```

```typescript
// Programmatic usage
import { varg } from 'varg'

// Start job
const job = await varg.run('kling', { 
  prompt: 'a cat dancing' 
})

// Subscribe to updates
job.on('progress', (p) => {
  console.log(`${p.percent}% - ${p.message}`)
})

job.on('done', (result) => {
  console.log(result.output)
})

// Or just wait
const result = await job.wait()

// Or one-liner
const result = await varg.run('flux', { prompt: 'cat' }).wait()
```

---

## Quick Reference

```bash
# Run
varg run <name> [options]       # run model/action/skill
varg run <name> --help          # show help
varg run <name> --schema        # show JSON schema
varg run <name> --dry-run       # preview execution
varg run <name> --watch         # follow progress
varg run <name> --wait          # block until done

# Jobs
varg jobs                       # list all jobs
varg jobs <id>                  # job status
varg jobs <id> --watch          # follow job
varg jobs <id> --logs           # show logs
varg jobs <id> --kill           # cancel job
varg jobs <id> --retry          # retry failed
varg jobs clean                 # remove completed

# Discover
varg list                       # list everything
varg list models|actions|skills # filter by type
varg find "<query>"             # fuzzy search
varg which <action>             # show routing

# Skills
varg skills                     # list skills
varg skills run <name>          # execute skill
varg skills create              # new skill
varg skills show <name>         # show definition

# Other
varg ai "<prompt>"              # natural language
varg schemas                    # export all schemas
```

---

<sub>varg v0.1 · made with ♥ by varg.ai</sub>
