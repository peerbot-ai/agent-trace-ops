# Token optimization plugin for Claude Code

Analyzes your Claude Code conversation history, enriches tool call metadata, compresses conversation data with RLE, and uses Claude Code to detect repetitive multi-step patterns. It suggests:
1. Helper scripts that trigger multiple tool calls in a single Bash command to save tokens on repeated workflows.
2. File refactorings that merges/splits files to save tokens on repeated workflows.

![Demo](demo.gif)

## Usage

### Claude Code Plugin

```bash
/plugin marketplace add peerbot-ai/claude-code-optimizer
/plugin install agent-trace-ops
```

After installing the plugin, use the `/plan` command for on-demand analysis:

```bash
/agent-trace-ops:plan
```

### CLI

```bash
npx agent-trace-ops
```
Install globally for repeated use:

```bash
npm install -g agent-trace-ops
ato --project-path=<path> --agent=claude
```

This will:
1. Check for existing analysis reports
2. Ask if you want to reuse or regenerate the report
3. Let you select which optimization categories to analyze:
   - Quick Commands (one-liner chains for package.json/Makefile)
   - Parameterized Scripts (reusable workflows)
   - File Refactorings (merge/split frequently accessed files)
4. Launch parallel Task agents to analyze patterns
5. Generate helpers and show potential token savings

## How it works

### Claude Session Files

Every conversation in Claude Code is saved as JSONL (JSON Lines) files in `~/.claude/projects/<hash>/`. Here's what a real session looks like:

```jsonl
{"type":"text","text":"Let me search for authentication code"}

{"type":"tool_use","id":"toolu_01ABC","name":"codebase_search","input":{
  "query":"How does user authentication work?",
  "target_directories":[]
}}

{"type":"thinking","thinking":"I found the auth flow in backend/auth/. Now I should look at the middleware to see how sessions are validated..."}

{"type":"tool_use","id":"toolu_02DEF","name":"read_file","input":{
  "target_file":"backend/auth/middleware.js"
}}

{"type":"tool_use","id":"toolu_03GHI","name":"read_file","input":{
  "target_file":"backend/auth/session.js"
}}

{"type":"tool_use","id":"toolu_04JKL","name":"read_file","input":{
  "target_file":"backend/auth/validators.js"
}}
```

### What This Tool Does

**agent-trace-ops** analyzes these session files to find patterns:

1. 🔍 **Detects repetitive workflows** - Spots when you read the same files together multiple times
2. 🧠 **Includes thinking blocks** - Uses Claude's internal reasoning to understand *why* actions were taken
3. 📊 **Compresses with RLE** - Groups repeated patterns: `[read_file × 3]` instead of listing each call
4. 🤖 **Sends to Claude AI** - Let's Claude analyze the patterns and suggest optimizations

The thinking blocks are especially valuable because they reveal the *intent* behind tool calls:
> "I should look at the middleware to see how sessions are validated"

This helps Claude AI suggest better optimizations like:
- Merge `middleware.js`, `session.js`, and `validators.js` → `auth/core.js`
- Create script: `./scripts/show-auth-flow.sh` to display all auth files at once

### Example Optimizations

**Quick Commands** - Combine repeated bash sequences:
```bash
# Before: 4 separate Bash calls each time you test
npm run build
npm run test
npm run lint
npm run format:check

# After: 1 command in package.json scripts
npm run precommit  # Reduces from 4 calls to 1
# package.json: "precommit": "npm run build && npm test && npm run lint && npm run format:check"
```

**Parameterized Scripts** - Reusable workflows:
```bash
# Before: 4 Bash calls every time you debug a service
docker ps | grep auth-service
docker logs auth-service --tail=100
docker exec auth-service cat /app/config.json
docker stats auth-service --no-stream

# After: 1 script call with parameters
./scripts/debug-service.sh auth-service 100  # Reduces from 4 calls to 1
# Reusable for any service: api-gateway, payment-processor, etc.
```

**File Refactorings** - Merge co-accessed files:
```bash
# Before: Reading 3 files per change
1. Read: [+0s 6156b] types.ts[L1-L203]
2. Read: [+1s 4720b] utils.ts[L1-L156]
3. Read: [+1s 2696b] config.ts[L1-L89]

# After: Merged into src/core.ts (448 lines)
1. Read: [+0s 13572b] core.ts[L1-L448]  # Reduces from 3 reads to 1
```

> **See [instructions.md](instructions.md) for detailed pattern examples and optimization strategies.**

## Development

```bash
# Clone the repository
git clone https://github.com/peerbot-ai/agent-trace-ops.git
cd agent-trace-ops

# Run locally
node index.js

# Test hook mode
node index.js --format=hook
```

## Publishing

### To npm

```bash
npm publish
```