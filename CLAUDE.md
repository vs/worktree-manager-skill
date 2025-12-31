# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Claude Code skill for managing parallel development environments using git worktrees. The skill automates worktree creation, dependency installation, port allocation, and launching Claude Code agents in separate terminal windows.

**Core concept**: All worktrees are stored centrally at `~/tmp/worktrees/` and tracked in a global registry at `~/.claude/worktree-registry.json`. Each worktree gets unique ports allocated from a global pool (8100-8199) to avoid conflicts across all projects.

## Architecture

### File Structure

This repository follows the Claude Code marketplace plugin structure:

```
.
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest for marketplace
│   └── marketplace.json     # Marketplace catalog for distribution
├── skills/
│   └── worktree-manager/    # The skill itself
│       ├── SKILL.md         # Comprehensive skill documentation for Claude
│       ├── config.json      # Skill configuration (terminal, shell, port pool)
│       ├── scripts/         # Helper bash scripts (all operations can be done manually)
│       │   ├── allocate-ports.sh    # Allocate N ports from global pool
│       │   ├── register.sh          # Add worktree entry to global registry
│       │   ├── launch-agent.sh      # Open terminal window with Claude Code
│       │   ├── status.sh            # Show all worktrees (globally or per-project)
│       │   ├── cleanup.sh           # Remove worktree, release ports, optionally delete branch
│       │   └── release-ports.sh     # Release ports back to pool
│       └── templates/
│           └── worktree.json        # Optional project-specific config schema
├── CLAUDE.md                # This file - architecture guide for Claude Code
├── README.md                # User-facing documentation
└── cover.png                # Skill cover image
```

**Installation location:**
`~/.claude/plugins/marketplaces/worktree-manager-marketplace/`

### Global Registry Schema

Location: `~/.claude/worktree-registry.json`

```json
{
  "worktrees": [
    {
      "id": "uuid",
      "project": "project-name",
      "repoPath": "/path/to/original/repo",
      "branch": "feature/auth",
      "branchSlug": "feature-auth",
      "worktreePath": "~/tmp/worktrees/project-name/feature-auth",
      "ports": [8100, 8101],
      "createdAt": "2025-12-31T10:00:00Z",
      "validatedAt": "2025-12-31T10:02:00Z",
      "agentLaunchedAt": "2025-12-31T10:03:00Z",
      "task": "Implement OAuth login",
      "prNumber": null,
      "status": "active"
    }
  ],
  "portPool": {
    "start": 8100,
    "end": 8199,
    "allocated": [8100, 8101]
  }
}
```

### Branch Slug Convention

Branch names are slugified for filesystem paths by replacing `/` with `-`:
- `feature/auth` → `feature-auth`
- `fix/login-bug` → `fix-login-bug`

Command: `echo "feature/auth" | tr '/' '-'`

## Common Development Commands

### Testing Scripts

All scripts are located in `skills/worktree-manager/scripts/`:

```bash
# Test port allocation
./skills/worktree-manager/scripts/allocate-ports.sh 2

# Test registry operations
cat ~/.claude/worktree-registry.json | jq '.worktrees[]'

# Test launch agent (requires an existing worktree path)
./skills/worktree-manager/scripts/launch-agent.sh ~/tmp/worktrees/some-project/some-branch "Test task"

# Test status
./skills/worktree-manager/scripts/status.sh
./skills/worktree-manager/scripts/status.sh --project worktree-manager-skill

# Test cleanup
./skills/worktree-manager/scripts/cleanup.sh project-name feature/branch-name

# Validate plugin structure (if claude CLI is installed)
claude plugin validate .
```

### Manual Operations (when scripts fail)

**Initialize empty registry:**
```bash
mkdir -p ~/.claude
cat > ~/.claude/worktree-registry.json << 'EOF'
{
  "worktrees": [],
  "portPool": {
    "start": 8100,
    "end": 8199,
    "allocated": []
  }
}
EOF
```

**Find available ports:**
```bash
ALLOCATED=$(cat ~/.claude/worktree-registry.json | jq -r '.portPool.allocated[]' | sort -n)
for PORT in $(seq 8100 8199); do
  if ! echo "$ALLOCATED" | grep -q "^${PORT}$"; then
    if ! lsof -i :"$PORT" &>/dev/null; then
      echo "Available: $PORT"
      break
    fi
  fi
done
```

**Add worktree to registry manually:**
```bash
TMP=$(mktemp)
jq '.worktrees += [{
  "id": "'$(uuidgen)'",
  "project": "my-project",
  "repoPath": "/path/to/repo",
  "branch": "feature/auth",
  "branchSlug": "feature-auth",
  "worktreePath": "~/tmp/worktrees/my-project/feature-auth",
  "ports": [8100, 8101],
  "createdAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
  "validatedAt": null,
  "agentLaunchedAt": null,
  "task": "My task",
  "prNumber": null,
  "status": "active"
}]' ~/.claude/worktree-registry.json > "$TMP" && mv "$TMP" ~/.claude/worktree-registry.json
```

**Query registry:**
```bash
# All worktrees
cat ~/.claude/worktree-registry.json | jq '.worktrees[]'

# Specific project
cat ~/.claude/worktree-registry.json | jq '.worktrees[] | select(.project == "my-project")'

# Find by branch
cat ~/.claude/worktree-registry.json | jq '.worktrees[] | select(.branch | contains("auth"))'
```

## Key Architectural Patterns

### Centralized Storage
All worktrees live in `~/tmp/worktrees/<project>/<branch-slug>/` regardless of which project they belong to. This enables:
- Global port management across all projects
- Centralized cleanup and status checks
- Consistent agent launching

### Global Port Pool
Port allocation is global to prevent conflicts when running dev servers across multiple worktrees in different projects:
- Pool: 8100-8199 (100 ports)
- Default allocation: 2 ports per worktree (for API + frontend patterns)
- Ports are verified against both registry and system (`lsof`)

### Script Fallback Pattern
All scripts are optional helpers. Claude can perform any operation manually using `jq`, `git`, and bash commands. This is documented extensively in SKILL.md under "Manual Operations" sections.

### Package Manager Detection
Detect by checking for lockfiles in priority order:
1. `bun.lockb` → `bun install`
2. `pnpm-lock.yaml` → `pnpm install`
3. `yarn.lock` → `yarn install`
4. `package-lock.json` → `npm install`
5. `uv.lock` → `uv sync`
6. `pyproject.toml` → `uv sync`
7. `requirements.txt` → `pip install -r requirements.txt`
8. `go.mod` → `go mod download`
9. `Cargo.toml` → `cargo build`

### Parallel Worktree Creation
When creating multiple worktrees:
1. Allocate all ports upfront
2. Use subagents for parallel creation
3. Each subagent: create → install deps → validate → register → launch
4. Report unified summary with any failures

## Project-Specific Configuration

Projects can optionally provide `.claude/worktree.json` for custom settings (see `skills/worktree-manager/templates/worktree.json` for schema). This overrides default behavior for:
- Port count and service names
- Custom install commands
- Validation start/stop/healthcheck commands
- Directories to copy from original repo

## Terminal Support

The skill supports multiple terminals via `~/.claude/plugins/marketplaces/worktree-manager-marketplace/skills/worktree-manager/config.json`:
- `ghostty` (default)
- `iterm2`
- `tmux`
- `wezterm`
- `kitty`
- `alacritty`

Each has different launch syntax implemented in `skills/worktree-manager/scripts/launch-agent.sh:84-144`.

## Requirements

- `jq` - Required for all registry operations
- `git` - Required for worktree commands
- Configured terminal application
- `lsof` - Used for port availability checks

## Important Notes

1. **Scripts are helpers, not requirements**: Every operation can be done manually with standard Unix tools. `skills/worktree-manager/SKILL.md` documents both approaches.

2. **Global registry is source of truth**: Always check/update `~/.claude/worktree-registry.json` for port allocations and worktree tracking.

3. **Port safety**: Always verify port isn't in use by system (`lsof -i :PORT`) before allocating, even if registry shows it's free.

4. **Slugification**: Use `tr '/' '-'` to convert branch names to filesystem-safe paths.

5. **Worktree paths**: Always use absolute paths. Expand `~` to `$HOME` when needed.
