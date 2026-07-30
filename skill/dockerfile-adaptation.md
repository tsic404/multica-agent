---
name: dockerfile-adaptation
description: Dockerfile adaptation workflow for docker-images project. Fetcher pattern, version.sh rules, upstream diff analysis, git workflow. Load when Dockerfile agent handles [sync] or 适配 mode issues.
allowed-tools: Bash(git:*), Bash(curl:*), Bash(docker:*), Bash(grep:*)
---

# Dockerfile Adaptation Workflow

Adapt upstream Dockerfiles to docker-images fetcher pattern.

## Fetcher Pattern (Universal)

All docker-images projects use two-stage build:

**Stage 1 — Fetcher**: git clone source code
```dockerfile
FROM ubuntu:latest AS fetcher
ARG VERSION=<default>
WORKDIR /workspace
RUN apt-get update && apt-get install -y git curl && rm -rf /var/lib/apt/lists/*
RUN git clone --branch ${VERSION} --depth=1 <repo_url> .
```

**Stage 2 — Runtime**: build + run
```dockerfile
FROM <base_image> AS <project_name>
ARG VERSION=<default>
RUN apt-get update && apt-get install -y <deps> && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY --from=fetcher /workspace/<source_dir>/ ./
RUN <npm/pip/go build>
EXPOSE <port>
CMD [...]
```

## Workflow

### Step 1: Load project info from `docker-upstream-dockerfile-sync` skill

### Step 2: Fetch upstream reference
```bash
curl -sL "https://raw.githubusercontent.com/<owner>/<repo>/<branch>/<path>" -o /tmp/upstream.Dockerfile
# Node.js: also fetch package.json
```

### Step 3: Analyze diff

- **Expected differences (KEEP)**: fetcher stage, COPY --from=fetcher, our enhancements
- **Must sync**: new system deps, npm/pip package changes, port changes, CMD/ENTRYPOINT changes, new COPY paths
- **Uncertain**: mark as `[待确认]`, STOP and list — don't guess

### Step 4: Adapt — map upstream to fetcher

| Upstream | Our Pattern |
|----------|------------|
| `COPY package.json .` | `COPY --from=fetcher /workspace/<repo>/package.json ./` |
| `COPY src/ ./src/` | `COPY --from=fetcher /workspace/<repo>/src/ ./src/` |
| `RUN npm install` | Keep, but COPY package.json first for cache |
| `RUN --mount=type=bind,...` | Convert to fetcher-stage curl or git clone |

### Step 5: Handle version.sh

**LAST_VERSION rule**: NOT the software version — it's the "last CI-built version". CI triggers build when `current_version != LAST_VERSION`.

| Scenario | LAST_VERSION |
|----------|-------------|
| New project | Set to `""` or `"0"` |
| Existing project | **NEVER modify** — even if ARG VERSION default looks newer |

### Step 6: Verify
```bash
grep -E "^FROM|^COPY|^RUN|^CMD|^ENTRYPOINT|^EXPOSE" /workspace/docker-images/<project>/Dockerfile
```

## Git Workflow

Always work on feature branches:
```bash
git checkout master && git pull origin master
git checkout -b <project>-<desc>
```

Commit format: `type: head` + body (type: feat/fix/sync/chore)
```bash
git add <project>/Dockerfile <project>/version.sh
git commit -m "sync: <project> 适配上游 <summary>"
git fetch origin master && git rebase origin/master
git push origin HEAD:master
```

## Forbidden Actions

| Forbidden | Reason |
|-----------|--------|
| ❌ Copy/replace upstream Dockerfile directly | Loses fetcher stage |
| ❌ Rewrite COPY to bare `COPY .` | No source in build context |
| ❌ Suggest "remove fetcher stage" | Universal pattern for all projects |
| ❌ Modify LAST_VERSION on existing projects | CI relies on it for build triggering |
| ❌ Delete or overwrite version.sh | Version detection depends on it |
| ❌ Modify on master directly | Must use feature branch |
| ❌ Skip `rm -rf /var/lib/apt/lists/*` after apt-get | Image bloat |

## Output

After completion, report:
1. Modified files + diff summary
2. Change rationale (which upstream change drove it)
3. version.sh changes; LAST_VERSION only modified for new projects
4. Any `[待确认]` decisions for human review
