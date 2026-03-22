---
name: githuman
description: 'How to use GitHuman to review AI-generated code changes before committing. GitHuman provides a GitHub-like diff review interface for your staging area — visual diffs, inline comments, todos, and review workflows — all running locally via Docker. Use this skill whenever the user mentions "githuman", "code review before commit", "review staged changes", "review AI changes", wants to review diffs in a browser, asks about reviewing code before committing, or after completing a coding task where staged changes should be reviewed. Also trigger when you see references to sjust/ajust githuman commands.'
---

# GitHuman

GitHuman moves the code review checkpoint from "after push" to "before commit". It gives you a GitHub-like interface to review staged changes — visual diffs with syntax highlighting, inline comments with code suggestions, todo tracking, and a full review workflow — all running locally in Docker containers.

This environment runs GitHuman through Docker containers managed by Just recipes. The command runner is platform-specific:

- **macOS** (default): `sjust`
- **Linux**: `ajust`

Throughout this skill, commands use `$JUST_CMD` as a placeholder. Always detect the platform first — macOS is the primary OS, so default to `sjust` unless you are certain the user is on Linux:

```bash
if [[ "$(uname)" == "Linux" ]]; then
    JUST_CMD="ajust"
else
    JUST_CMD="sjust"
fi
```

When providing commands to the user, substitute the correct command (`sjust` or `ajust`) based on their OS. When in doubt, use `sjust`.

## Available commands

All commands belong to the `githuman` group in Just.

### Start a review instance

```bash
$JUST_CMD githuman-start [directory]
```

Launches a new GitHuman Docker container for the given project directory (defaults to `.`). The command:

1. Checks for an existing instance on the same directory (reuses it if found)
2. Generates a unique container name and FQDN (`<project>.githuman.sparkfabrik.loc`)
3. Handles name collisions with numeric suffixes when multiple instances share a basename
4. Creates a random auth token for the session
5. Provisions TLS via `spark-http-proxy` if available
6. Waits for the container to become ready (up to 60s)
7. Opens the review URL in the default browser

The resulting URL looks like: `https://<project>.githuman.sparkfabrik.loc?token=<token>`

### Open a running instance

```bash
$JUST_CMD githuman-open [directory]
```

Opens the browser for an already-running GitHuman instance on the given directory. If no instance exists, starts a new one automatically.

### Get the container ID

```bash
$JUST_CMD githuman-id [directory]
```

Prints the Docker container ID for the GitHuman instance running on the given directory. Useful for scripting or debugging.

### List all instances

```bash
$JUST_CMD githuman-list
```

Shows all running GitHuman containers with their name, project directory, and URL.

### View logs

```bash
$JUST_CMD githuman-logs [container-name]
```

Streams logs for a GitHuman container. If no name is given, presents an interactive selection menu.

### Stop an instance

```bash
$JUST_CMD githuman-stop [container-name]
```

Stops a specific GitHuman container. Without a name, presents an interactive menu with a "Stop all" option.

### Purge everything

```bash
$JUST_CMD githuman-purge
```

Stops all GitHuman containers and removes the shared Docker volumes (npm cache and review data). This is a destructive operation — review data is lost.

## Review workflow

The typical review cycle when working with AI coding agents:

1. **AI agent makes changes** — the agent writes or modifies code as part of a task.
2. **Stage the changes** — run `git add` on the files to review (the agent can do this, or the user can stage selectively).
3. **Suggest a review** — after completing a coding task, suggest launching GitHuman so the user can review the changes before committing. Example: *"I've staged the changes. Would you like to review them in GitHuman? Run `sjust githuman-start` (macOS) or `ajust githuman-start` (Linux) to open the review interface."* Do not launch it automatically.
4. **User reviews in the browser** — the GitHuman interface shows:
   - File-by-file diffs with syntax highlighting
   - Inline commenting on specific lines (with optional code suggestions)
   - Review status tracking (in progress, approved, changes requested)
5. **Act on feedback** — if the user reports issues from the review or requests changes, address them and re-stage.
6. **Commit when approved** — once the review is approved, the user commits the changes.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Container not starting | Check Docker is running: `docker info` |
| Review not ready after 60s | Check logs: `$JUST_CMD githuman-logs` |
| Certificate warning in browser | Install `spark-http-proxy` or accept the self-signed cert |
| Port conflict | GitHuman uses port 3847 inside the container; the reverse proxy handles external routing |
| Stale instance | Stop and restart: `$JUST_CMD githuman-stop` then `$JUST_CMD githuman-start` |
