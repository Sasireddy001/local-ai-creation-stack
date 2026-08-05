# Ruflo — Automated Repository Setup, Source-Code Audit & Architecture Analysis

- **Source repo:** `https://github.com/ruvnet/ruflo.git` (also `https://github.com/ruvnet/claude-flow.git`)
- **Local clone:** `local-ai-creation-stack/ruflo/`
- **Audited version:** `ruflo@3.34.0` / `@claude-flow/cli@3.34.0` (package manifests)
- **User Guide version referenced:** `3.7.0-alpha.8`
- **Audit environment:** Windows, Node `v20.20.2`, npm `10.8.2`, Rust/Cargo not installed on this host

---

## Phase 1: Environment Initialization & Clone Verification

### 1.1 Clone state
```bash
cd local-ai-creation-stack/ruflo
git status
# On branch main, up to date with origin/main, working tree clean
```

### 1.2 Runtime presence on audit host
| Tool | Version / Status |
|---|---|
| Node.js | `v20.20.2` |
| npm | `10.8.2` |
| pnpm | not installed |
| cargo | not installed |
| rustup | not installed |

> The repository's TypeScript/Node.js stack can be exercised immediately. Rust components (`v3/crates/*`) require `cargo` and are not compiled here, but the Cargo workspace manifest was inspected.

---

## SECTION 1: Exact Repository Contents & System Architecture

### 1.1 Directory structure
```
ruflo/
├── bin/                              # Umbrella CLI entry points
│   ├── cli.js                        # Proxies to v3/@claude-flow/cli/bin/cli.js
│   ├── npx-repair.js
│   └── npx-safe-launch.js
├── ruflo/                            # Published `ruflo` npm package root
│   ├── bin/ruflo.js                  # Public `ruflo` binary
│   ├── src/mcp-bridge/               # White-label chat UI + MCP bridge
│   ├── docker-compose.yml
│   ├── docker-compose.public.yml
│   ├── rvf.manifest.json             # RVF package manifest
│   └── package.json
├── v3/@claude-flow/cli/              # Core CLI package
│   ├── bin/cli.js                    # Main `claude-flow` / `ruflo` binary
│   ├── bin/mcp-server.js             # Dedicated stdio MCP server
│   ├── src/index.ts                  # CLI class (830 lines)
│   ├── src/mcp-server.ts             # MCPServerManager (stdio/HTTP/WebSocket)
│   ├── src/mcp-tools/                # 30+ MCP tool modules
│   │   ├── index.ts                  # Re-exports all tool families
│   │   ├── agent-tools.ts
│   │   ├── swarm-tools.ts
│   │   ├── memory-tools.ts
│   │   ├── hooks-tools.ts
│   │   ├── autopilot-tools.ts
│   │   ├── embeddings-tools.ts
│   │   └── ...
│   ├── src/swarm/
│   ├── src/memory/
│   ├── src/ruvector/
│   ├── src/security/
│   └── package.json
├── v3/@claude-flow/cli-core/         # Lightweight memory-only fast path
├── v3/@claude-flow/codex/
├── v3/@claude-flow/plugin-agent-federation/
├── v3/@claude-flow/security/
├── v3/crates/ruflo-federation-peer/  # Rust federation peer
├── v3/crates/ruflo-agntcy/           # Rust agntcy bindings
├── v3/plugins/gastown-bridge/        # Excluded Cargo workspace (WASM)
├── plugins/                          # 33 installable plugins
│   ├── ruflo-core/
│   ├── ruflo-agentdb/
│   ├── ruflo-aidefence/
│   ├── ruflo-autopilot/
│   ├── ruflo-intelligence/
│   ├── ruflo-ruvector/
│   ├── ruflo-swarm/
│   └── ...
├── docs/                             # User guide, ADRs, status
│   ├── USERGUIDE.md
│   ├── STATUS.md
│   └── adr/
├── AGENTS.md                         # Agent registry & metadata
├── SECURITY.md
├── Cargo.toml                        # Root workspace manifest
├── package.json                      # Root `claude-flow` package
└── tsconfig.json
```

### 1.2 Engine components & stack

| Layer | Component | Technology | Key files |
|---|---|---|---|
| CLI shell | `ruflo` / `claude-flow` binary | Node.js ESM, `commander`, `inquirer` | `bin/cli.js`, `v3/@claude-flow/cli/bin/cli.js` |
| CLI core | `CLI` class, parser, command registry | TypeScript | `v3/@claude-flow/cli/src/index.ts`, `parser.ts`, `commands/index.ts` |
| MCP server | JSON-RPC stdio/HTTP/WebSocket | Node.js | `v3/@claude-flow/cli/bin/mcp-server.js`, `src/mcp-server.ts` |
| MCP tools | 323 tools in families | TypeScript | `v3/@claude-flow/cli/src/mcp-tools/index.ts` |
| Swarm coordination | Agent spawn, topologies, consensus | TypeScript | `v3/@claude-flow/cli/src/mcp-tools/swarm-tools.ts`, `v3/@claude-flow/cli/src/mcp-tools/agent-tools.ts` |
| Memory/RAG | AgentDB, RuVector, HNSW, SQLite | `agentdb`, `better-sqlite3`, ONNX | `plugins/ruflo-agentdb/README.md`, `v3/@claude-flow/cli/src/mcp-tools/embeddings-tools.ts` |
| Neural learning | SONA, ReasoningBank, EWC++, MicroLoRA | `@claude-flow/neural` | `plugins/ruflo-intelligence/README.md` |
| Security | AIDefence, policy engine, encryption | `@claude-flow/security`, `agentdb` | `plugins/ruflo-aidefence/README.md`, `v3/@claude-flow/cli/src/security/` |
| Federation | Peer-to-peer agent comms | Rust + TypeScript bindings | `v3/crates/ruflo-federation-peer/`, `v3/@claude-flow/plugin-agent-federation/` |
| Chat UI | White-label HuggingFace Chat UI | SvelteKit, Docker | `ruflo/src/ruvocal/`, `ruflo/docker-compose.yml` |
| WASM kernels | Embeddings, policy, proof | Rust → WASM | `@ruvector/rabitq-wasm`, optional `gastown-bridge` |

### 1.3 Dependency & environment requirements

#### Node.js / npm
- **Required:** Node.js `>=20.0.0` (from `package.json` `engines` and `ruflo/package.json` `engines`)
- **TypeScript:** `^5.0.0` (`devDependencies`)
- **Build runner:** `tsx`, `vitest`

#### Core npm dependencies
| Package | Purpose |
|---|---|
| `@claude-flow/cli-core` | Fast memory-only lite path |
| `@claude-flow/cli` | Full orchestration CLI |
| `@claude-flow/mcp` | MCP protocol helpers |
| `@claude-flow/neural` | SONA / ReasoningBank / EWC++ |
| `@claude-flow/security` | Policy + encryption |
| `@claude-flow/codex` | OpenAI Codex integration |
| `@claude-flow/plugin-agent-federation` | Federation peer |
| `@ruvector/rabitq-wasm` | 1-bit quantized embeddings |
| `commander`, `inquirer`, `chalk`, `zod` | CLI UX |
| `bcryptjs`, `@noble/ed25519` | Crypto / verification |

#### Optional native dependencies
| Package | Purpose |
|---|---|
| `better-sqlite3` `12.9.0` | Local SQLite persistence |
| `agentdb` `^3.0.0-alpha.17` | AgentDB memory bridge |
| `ruvector` `^0.2.27` | ONNX embedding/RuVector engine |
| `@napi-rs/keyring` | OS keychain access |

#### Rust / Cargo
- Workspace members: `v3/crates/ruflo-federation-peer`, `v3/crates/ruflo-agntcy`
- Excluded nested workspace: `v3/plugins/gastown-bridge` (WASM)
- Manifest: `Cargo.toml` (resolver `2`)

---

## SECTION 2: Functional Scope & Execution Capabilities

### 2.1 Swarm & multi-agent mechanics

#### Agent and swarm tool families
| Family | Tool count | Source file |
|---|---|---|
| `agent_*` | 8 | `v3/@claude-flow/cli/src/mcp-tools/agent-tools.ts` |
| `swarm_*` | 4 | `v3/@claude-flow/cli/src/mcp-tools/swarm-tools.ts` |

Exposed capabilities (`plugins/ruflo-swarm/README.md`):
- `agent_spawn`, `agent_execute`, `agent_terminate`, `agent_status`, `agent_list`, `agent_pool`, `agent_health`, `agent_update`
- `swarm_init`, `swarm_status`, `swarm_shutdown`, `swarm_health`

#### Topologies
| Topology | Use case |
|---|---|
| `hierarchical` | Queen coordinator + workers (default anti-drift) |
| `mesh` | Peer-to-peer |
| `hierarchical-mesh` | Queen + peer comms for 10+ agents |
| `ring` | Round-robin task handoff |
| `star` | Central hub |
| `adaptive` | Runtime topology switching |

#### Consensus strategies
- `raft` (leader-elected, default)
- `byzantine` (`f < n/3`, 2/3 majority)
- `gossip` (eventually consistent)
- `crdt` (conflict-free replicated data type)
- `quorum` (configurable threshold)

#### Anti-drift canonical defaults
```json
{
  "topology": "hierarchical",
  "maxAgents": "6-8",
  "strategy": "specialized",
  "consensus": "raft",
  "memory": "hybrid"
}
```
Source: `plugins/ruflo-swarm/README.md`.

### 2.2 Model Context Protocol (MCP) & local CLI integration

#### MCP server entry points
```bash
npx ruflo@latest mcp start                 # explicit stdio
npx @claude-flow/cli@latest                # auto-detects MCP when stdin piped
cat input.json | npx ruflo                 # stdio JSON-RPC mode
```

#### MCP protocol implementation
- **File:** `v3/@claude-flow/cli/bin/mcp-server.js` and `bin/cli.js`
- **Protocol version:** `2024-11-05`
- **Server info:** `{ name: 'ruflo', version: '3.0.0' }`
- **Capabilities:** `tools.listChanged`, `resources.subscribe`, `resources.listChanged`
- **Methods handled:** `initialize`, `tools/list`, `tools/call`, `notifications/initialized`, `ping`
- **DoS cap:** `MCP_MAX_BUFFER_BYTES = 10 * 1024 * 1024` (10 MB stdin buffer)

#### Tool advertisement filtering
```bash
ruflo mcp start --tools memory,agentdb,swarm
ruflo mcp start --tools=agent_*
env CLAUDE_FLOW_MCP_TOOLS=memory,agentdb ruflo mcp start
```
Filter logic: `v3/@claude-flow/cli/bin/cli.js:41-60` and `v3/@claude-flow/cli/src/mcp-server.ts:82-107`.

#### MCP tool surface (verified counts)
| Surface | Count | Source |
|---|---|---|
| MCP tools | **323** | `docs/STATUS.md` + `verification/inventory.json` |
| CLI top-level commands | **45** | `docs/STATUS.md` |
| Plugins | **33** | `plugins/ruflo-*/` directories |
| Agent definitions | **45** | `plugins/*/agents/*.md` |

Core plugin `ruflo-core` advertises **314 tools** directly (memory, agentdb, embeddings, hooks, neural, autopilot, browser, aidefence, agent, swarm, system, terminal, github, daa, coordination, performance, workflow, etc.).

### 2.3 Input/output & routing specs

#### Request flow
```
User / Claude Code
  → MCP server / CLI parser
  → AIDefence security gate
  → Q-Learning / MoE / Skills / Hooks router
  → Swarm topology + consensus
  → Specialized agents
  → AgentDB / RuVector memory
  → LLM providers (Claude/GPT/Gemini/Ollama)
```
Source: `docs/USERGUIDE.md` architecture diagram.

#### 3-tier model routing (now Thompson-sampling bandit)
- Tiers: Haiku / Sonnet / Opus
- `hooks_model-outcome` updates Beta(α, β) priors per tier.
- `hooks_model-route` samples θ ~ Beta(α, β) and picks argmax.
- Cost: ~45 µs per route call.
- Source: `docs/USERGUIDE.md` "What's new in 3.7".

#### Intelligence pipeline
| Phase | Tools |
|---|---|
| RETRIEVE | `hooks_intelligence_pattern-search`, `agentdb_pattern-search`, `agentdb_semantic-route` |
| JUDGE | `hooks_intelligence_attention`, `neural_predict`, `hooks_explain` |
| DISTILL | `ruvllm_sona_adapt`, `ruvllm_microlora_adapt`, `neural_train`, `hooks_intelligence_learn` |
| CONSOLIDATE | `agentdb_consolidate`, `ruvllm_microlora_adapt --consolidate`, `neural_compress` |

---

## SECTION 3: Performance, Token Optimization & Technical Advantages

### 3.1 Shared memory & context persistence

#### AgentDB / RuVector substrate
| Component | Function |
|---|---|
| `memory_*` tools | Store/search/recall/bridge across namespaces |
| `agentdb_*` tools | 15 tools for hierarchical / pattern / causal storage |
| `embeddings_*` tools | 10 tools: 384-dim ONNX `all-MiniLM-L6-v2`, HNSW, hyperbolic, RaBitQ |
| `ruvllm_hnsw_*` tools | WASM-backed HNSW pattern router |

#### Quantization & compression
- **RaBitQ 1-bit quantization:** 32× memory reduction (`plugins/ruflo-agentdb/README.md`)
- **Int8 quantization:** ~4× memory reduction (`docs/USERGUIDE.md`)
- **EWC++ consolidation:** prevents catastrophic forgetting on SONA / MicroLoRA adapters (`plugins/ruflo-intelligence/README.md`)

#### Namespace scheme
- Format: `<plugin-stem>-<intent>` (kebab-case)
- Reserved namespaces: `pattern`, `claude-memories`, `default`
- Source: `plugins/ruflo-agentdb/README.md`

### 3.2 Hardware & resource profiles

| Component | CPU | RAM | GPU | Notes |
|---|---|---|---|---|
| CLI core | Node single process | Low baseline | n/a | Fast path `cli-core` ~1.5 s cold-cache |
| RuVector ONNX | multi-core preferred | scales with corpus | optional (ONNX runtime) | 384-dim embeddings |
| HNSW search | single-digit ms | vector index in RAM | n/a | sub-millisecond retrieval |
| AgentDB / SQLite | low | low | n/a | `better-sqlite3` WAL mode |
| Neural training (SONA/MicroLoRA) | moderate | moderate | optional | adapters are small |

#### Token/cost optimization
- **Agent Booster WASM:** skips LLM for simple edits (<1 ms)
- **Token optimizer:** 30–50% token reduction via compression/caching
- **Cache-aware /loop heartbeat:** 270 s wake interval stays under 5-min prompt-cache TTL (`plugins/ruflo-autopilot/README.md`)
- **Lite CLI path:** `@claude-flow/cli-core` drops cold-cache `npx` from ~35 s to ~1.5 s (22.9× faster for plugin scripts)

### 3.3 Security & safety layer (AIDefence)

#### `ruflo-aidefence` MCP surface
6 tools: `aidefence_scan`, `aidefence_analyze`, `aidefence_stats`, `aidefence_learn`, `aidefence_is_safe`, `aidefence_has_pii` plus `transfer_detect-pii`.

#### Canonical 3-gate pattern
| Gate | Tool | When |
|---|---|---|
| 1. Pre-storage PII | `aidefence_has_pii` | Before any `memory_store` / AgentDB write |
| 2. Sanitization | `aidefence_scan` | Cookies, tokens, high-entropy blobs |
| 3. Prompt-injection | `aidefence_is_safe` | Before extracted text re-enters an LLM prompt |

Detected categories (`plugins/ruflo-aidefence/README.md`):
- Prompt injection (`ignore all previous instructions`)
- Role hijack (`you are now …`)
- Jailbreak markers (`DAN mode`, `developer mode`, `god mode`, `root mode`)

#### Runtime hardening (ADR-095 / ADR-096)
- **Loader-hijack denylist:** `validateEnv()` rejects `LD_PRELOAD`, `LD_LIBRARY_PATH`, `LD_AUDIT`, `DYLD_*`, `NODE_OPTIONS`, `NODE_PATH` at `terminal_create` boundary.
- **Restricted file modes:** `fs-secure.writeFileRestricted` uses `0600` files / `0700` dirs for sessions, terminals, memory.
- **Encryption at rest:** opt-in `CLAUDE_FLOW_ENCRYPT_AT_REST=1` uses AES-256-GCM with `RFE1` magic-byte sniff.
- **MCP DoS cap:** 10 MB stdin buffer cap in `bin/mcp-server.js` and `bin/cli.js`.

---

## SECTION 4: Actionable Practical Capabilities & Real-World Use Cases

### 4.1 Autonomous developer workflows

Ruflo can run the following end-to-end software-engineering tasks out-of-the-box (via MCP tools, CLI, or Claude Code hooks):

| Task | Entry point | Key tools/commands |
|---|---|---|
| Automated coding | `ruflo agent spawn -t coder` | `agent_spawn`, `agent_execute` |
| Code review | Agent `reviewer` | `agent_*`, `memory_search` |
| Unit test generation | `ruflo testgen` / `testgen_*` tools | `v3/@claude-flow/cli/src/mcp-tools/testgen-tools.ts` |
| Security scanning | `ruflo security scan` | `aidefence_scan`, `security_*` |
| Refactoring | `ruflo agent spawn -t optimizer` | `agent_execute` with `memory` context |
| Documentation | Agent `documenter` | `hooks_route`, `agent_execute` |
| CI/CD orchestration | `ruflo workflow run` | `workflowTools` from `mcp-tools/index.ts` |
| Browser automation | `ruflo browser` | `browser_*` + `browser_session_*` (28 tools) |
| Autonomous loops | `/autopilot` + `/loop` | `autopilot_*` (10 tools) |

### 4.2 Programmatic automation

#### CI/CD integration
```yaml
# GitHub Actions example
- name: Ruflo security scan
  run: npx ruflo@latest security scan --path .
- name: Ruflo verify build witness
  run: npx ruflo@latest verify
```

#### Git hooks
```bash
# .git/hooks/pre-commit
npx ruflo@latest aidefence scan --input "$(git diff --cached)"
```

#### Background daemon
`v3/@claude-flow/cli/src/index.ts:213-221` auto-starts a single-instance background daemon unless `RUFLO_DAEMON_AUTOSTART=0`.

#### Useful CLI commands
```bash
ruflo doctor --fix            # diagnose & repair
ruflo verify                  # Ed25519 signed witness verification
ruflo agent spawn -t coder --name api-worker
ruflo swarm init --topology hierarchical --max-agents 8
ruflo memory search --query "auth patterns"
ruflo hooks intelligence --status
```

---

## SECTION 5: Step-by-Step Local Deployment & Cascade Co-Working Blueprint

### 5.1 Local deployment verification blueprint

#### Step 1 — prerequisites
```bash
node --version   # >= 20.0.0
npm --version    # >= 10
# optional for Rust/WASM components: cargo + rustup
```

#### Step 2 — install Ruflo
```bash
# One-line install (recommended)
curl -fsSL https://cdn.jsdelivr.net/gh/ruvnet/ruflo@main/scripts/install.sh | bash

# Or npx wizard
npx ruflo@latest init --wizard
```

What `init --wizard` does (`docs/STATUS.md`):
- Writes `CLAUDE.md` with hooks and routing rules
- Registers the MCP server with Claude Code
- Seeds `.claude-flow/` with config + memory

#### Step 3 — start the MCP server
```bash
# Dedicated stdio server
npx ruflo@latest mcp start

# Or with tool filtering
npx ruflo@latest mcp start --tools memory,agentdb,swarm

# HTTP/WebSocket transport
npx ruflo@latest mcp start -t http --port 3000
```

#### Step 4 — verify health
```bash
ruflo doctor
ruflo doctor --fix
ruflo doctor -c encryption
mcp tool call mcp_status --json
mcp tool call agentdb_controllers --json
```

#### Step 5 — run smoke contracts
```bash
bash plugins/ruflo-core/scripts/smoke.sh       # expected 10 passed
bash plugins/ruflo-agentdb/scripts/smoke.sh    # expected 10 passed
bash plugins/ruflo-swarm/scripts/smoke.sh      # expected 11 passed
bash plugins/ruflo-aidefence/scripts/smoke.sh  # expected 10 passed
```

### 5.2 Cascade + Ruflo co-working blueprint

#### Separation of concerns
| Layer | Cascade (this IDE assistant) | Ruflo background agents |
|---|---|---|
| Active file edits | Direct code changes, inline diffs | Not required |
| Large-scale analysis | File-by-file inspection | `agent_spawn -t reviewer`, `swarm_init` for parallel audit |
| Testing | Run targeted commands | `ruflo testgen`, `autopilot` loops |
| Memory / RAG | Session-local context | `memory_store`, `memory_search_unified`, `agentdb_*` |
| Security | Manual review flags | `aidefence_scan`, `aidefence_is_safe` |
| CI/CD | Write workflow files | `ruflo verify`, `ruflo security scan` |

#### Recommended side-by-side workflow
1. **Cascade drives the editor:** edit, refactor, run tests, commit.
2. **Ruflo daemon auto-starts** on first `ruflo` command (`src/index.ts` auto-starts `ensureDaemonRunning` unless `RUFLO_DAEMON_AUTOSTART=0`).
3. **Spawn background agents for heavy tasks:**
   ```bash
   ruflo agent spawn -t reviewer --name reviewer-1 --task "review auth flow"
   ruflo agent spawn -t tester --name tester-1 --task "generate unit tests for src/utils"
   ```
4. **Use Ruflo memory to keep context cheap:**
   ```bash
   mcp tool call memory_store --json -- '{"namespace":"cascade-context","key":"plan","value":"..."}'
   mcp tool call memory_search_unified --json -- '{"query":"refactor plan"}'
   ```
5. **Run security gates before LLM context re-entry:**
   ```bash
   mcp tool call aidefence_has_pii --json -- '{"text":"..."}'
   mcp tool call aidefence_is_safe --json -- '{"text":"..."}'
   ```
6. **Persist learned patterns automatically:**
   - `post-task --train-neural` writes to `pattern` namespace.
   - `agentdb_consolidate` runs periodically for EWC++ consolidation.

### 5.3 Exact configuration files referenced

| File | Purpose |
|---|---|
| `package.json` | Root `claude-flow` package, workspaces, deps |
| `ruflo/package.json` | Published `ruflo` npm package |
| `v3/@claude-flow/cli/package.json` | Core CLI package, MCP bin registration |
| `Cargo.toml` | Rust workspace members |
| `tsconfig.json` | TypeScript compiler options |
| `ruflo/rvf.manifest.json` | Chat UI + MCP bridge RVF package manifest |
| `ruflo/docker-compose.yml` | White-label chat UI + MongoDB deployment |
| `docs/USERGUIDE.md` | Full command reference |
| `docs/STATUS.md` | Capability inventory and audit status |
| `AGENTS.md` | Agent registry |
| `SECURITY.md` | Security policy |
