# git-context
[![CI](https://github.com/KDM-cli/git-context/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/KDM-cli/git-context/actions/workflows/ci.yml)

**Know your Git state from inside Node.js.**

`git-context` is a zero-configuration, read-only TypeScript library that exposes the Git repository state around an application, script, deployment, migration, or CI job. It intentionally is not a complete Git wrapper.

## Installation

```bash
npm install gitcontext
```

Node.js 20 or later and the `git` executable are required.

## Quick Start

```ts
import { git } from "git-context";

const context = git();

console.log({
  branch: context.branch,
  commit: context.shortCommit,
  dirty: context.dirty,
});
```

`git()` finds the enclosing repository through Git itself, so it works seamlessly from nested directories, monorepo packages, and linked worktrees. It returns a fresh snapshot on every call so deployment checks never rely on stale clean/dirty state.

## Usage

### 1. Reading Repository Context

Call `git()` to read an immutable snapshot of the current repository:

```ts
import { git } from "git-context";

const context = git();

console.log(`Branch: ${context.branch}`);
console.log(`Commit: ${context.commit} (${context.shortCommit})`);
console.log(`Clean: ${!context.dirty}`);
console.log(`Author: ${context.author} <${context.email}>`);
console.log(`Root: ${context.root}`);

if (context.remote) {
  console.log(`Remote: ${context.remote} (${context.remoteUrl})`);
}
```

#### `GitContext` Properties

| Property | Type | Description |
|---|---|---|
| `branch` | `string \| null` | Current branch name, or `null` when `HEAD` is detached |
| `commit` | `string` | Full 40-character SHA of `HEAD` |
| `shortCommit` | `string` | Abbreviated SHA of `HEAD` (7+ characters) |
| `dirty` | `boolean` | `true` if tracked, staged, or untracked changes exist |
| `detached` | `boolean` | `true` if `HEAD` points directly at a commit rather than a branch |
| `author` | `string` | Author name of the `HEAD` commit |
| `email` | `string` | Author email of the `HEAD` commit |
| `root` | `string` | Absolute filesystem path to the repository root directory |
| `commitDate` | `string` | ISO 8601 commit timestamp of `HEAD` |
| `tag` | `string \| undefined` | Exact tag pointing at `HEAD`, if tagged |
| `ahead` | `number \| undefined` | Commits ahead of upstream tracking branch, if configured |
| `behind` | `number \| undefined` | Commits behind upstream tracking branch, if configured |
| `stashCount` | `number` | Number of stash entries saved in the repository |
| `mergeConflict` | `boolean` | `true` if unmerged files exist (merge conflict state) |
| `submodules` | `readonly SubmoduleInfo[]` | List of submodules with paths, commit SHAs, and dirty states |
| `remote` | `string \| undefined` | Preferred remote name (`origin` when present, otherwise first available) |
| `remoteUrl` | `string \| undefined` | URL of the preferred remote |

### 2. Checking Repository Presence

Check whether a directory is inside a Git repository before executing Git-dependent logic:

```ts
import { git } from "git-context";

if (git.isRepository()) {
  const context = git();
  console.log(`Running in Git repo at ${context.root}`);
} else {
  console.log("Not running inside a Git repository; skipping Git checks.");
}
```

### 3. Enforcing Safety Guards

Prevent accidental deployments, database migrations, or release publishing from dirty working trees, untracked changes, or wrong branches:

```ts
import { git } from "git-context";

// Require a specific branch (throws BranchMismatchError if mismatched or detached)
git.requireBranch("main");

// Require a completely clean working tree (throws DirtyRepositoryError if dirty)
git.assertClean();

// Enforce both in a single check
git.requireCleanBranch("main");

// Generic assertion combining multiple criteria:
git.assert({
  branch: ["main", "release"], // accept array of allowed branches
  clean: true,                 // require clean working tree
  detached: false,             // reject detached HEAD
  tag: true,                   // require HEAD to have an exact tag (or specific string like "v1.0.0")
  unpushed: true,              // require no unpushed commits ahead of upstream
});

// Proceed with sensitive task safely
await deploy();
```

### 4. Watch Mode (Event-Driven)

React to Git changes in real time (e.g. for development servers, watch scripts, or live status displays):

```ts
import { git } from "git-context";

const watcher = git.watch({ interval: 1000 });

// Fired whenever commit, branch, or dirty state changes
watcher.on("change", (newContext, oldContext) => {
  console.log(`Repository changed: ${oldContext.shortCommit} → ${newContext.shortCommit}`);
});

// Fired when working tree transitions from clean to dirty
watcher.on("dirty", (context) => {
  console.warn("Working tree has uncommitted changes!");
});

// Fired when working tree transitions back to clean
watcher.on("clean", (context) => {
  console.log("Working tree is now clean.");
});

// Stop polling when done
watcher.stop();
```

### 5. Configuration File Support

Define repository assertion rules in `.gitcontextrc.json` or `package.json#gitContext` and enforce them with a single line:

```json
{
  "$schema": "https://github.com/KDM-cli/git-context",
  "assertions": {
    "branch": "main",
    "clean": true,
    "detached": false,
    "unpushed": true
  },
  "format": "table"
}
```

```ts
import { git } from "git-context";

// Enforce rules defined in configuration:
git.assertFromConfig();

// Or load the parsed config object directly:
const config = git.loadConfig();
```

### 6. Inspecting Another Directory (`cwd`)

By default, `git-context` discovers the repository enclosing `process.cwd()`. Pass `{ cwd }` to any method to inspect a different repository, submodule, or workspace:

```ts
import { git } from "git-context";

const repoContext = git({ cwd: "/srv/release-checkout" });

git.requireCleanBranch("main", { cwd: "/srv/release-checkout" });
```

### 7. Error Handling

`git-context` throws dedicated, typed errors extending `GitContextError`:

```ts
import {
  git,
  GitContextError,
  DirtyRepositoryError,
  BranchMismatchError,
  DetachedHeadError,
  TagMismatchError,
  UnpushedCommitsError,
  RepositoryNotFoundError,
  GitExecutableNotFoundError,
} from "git-context";

try {
  git.assert({
    branch: "main",
    clean: true,
    tag: true,
  });
} catch (error) {
  if (error instanceof DirtyRepositoryError) {
    console.error("Please commit or stash changes before deploying.");
  } else if (error instanceof BranchMismatchError) {
    console.error(`Expected branch "${error.expected}", but found "${error.actual}".`);
  } else if (error instanceof DetachedHeadError) {
    console.error("HEAD is detached; checkout a named branch.");
  } else if (error instanceof TagMismatchError) {
    console.error(`Tag requirement failed (expected: ${error.expected}, actual: ${error.actual}).`);
  } else if (error instanceof UnpushedCommitsError) {
    console.error(`Please push ${error.ahead} local commit(s) before publishing.`);
  } else if (error instanceof RepositoryNotFoundError) {
    console.error(`No Git repository found from: ${error.cwd}`);
  } else if (error instanceof GitExecutableNotFoundError) {
    console.error("Git is not installed or not found on PATH.");
  } else if (error instanceof GitContextError) {
    console.error(`Git context error: ${error.message}`);
  }
}
```

### 8. Command Line Interface (CLI)

Run `git-context` directly from the terminal without installing:

```bash
npx git-context
```

#### Output Formats

- **Table (default)**: Color-coded, aligned table output displaying branch, commit, dirty status, author, remotes, tracking, stash count, merge conflict alerts, and submodules.
- **Minimal (`--format=minimal`)**: Compact one-liner with color highlights, ideal for terminal prompts and status lines (e.g., `main* a1b2c3d [1↑0↓]`).
- **JSON (`--json` or `--format=json`)**: Machine-readable JSON for CI/CD pipelines and scripting.

```bash
# Compact format
npx git-context --format=minimal

# Disable ANSI colors
npx git-context --no-color

# Output JSON
npx git-context --json
```

#### Scaffold Configuration (`init`)

Generate a `.gitcontextrc.json` configuration file and a sample `scripts/deploy-guard.ts` guard script:

```bash
npx git-context init
```

Flags:
- `--format=<format>`: Output format: `table` (default), `json`, `minimal`
- `--json`: Output machine-readable JSON (alias for `--format=json`)
- `--no-color`: Disable colored output
- `-h`, `--help`: Show usage and options
- `--version`: Show version number

## Examples

See the [examples/](examples/) directory:

- [basic.ts](examples/basic.ts) — read and print Git context
- [deployment.ts](examples/deployment.ts) — deploy only from clean `main`
- [migration.ts](examples/migration.ts) — run migrations with branch guard
- [express.ts](examples/express.ts) — embed Git context in HTTP response headers
- [ci-guard.ts](examples/ci-guard.ts) — CI pre-deploy validation
- [release.ts](examples/release.ts) — release script with metadata logging

## Design and Security

All Git commands are executed with Node's non-shell process API (`execFileSync`) using explicit argument arrays. `git-context`:
- Never interpolates arguments into a shell command (safe from shell injection)
- Never changes Git configuration or modifies the repository (read-only)
- Never contacts remote servers or performs network requests
- Emits zero telemetry
- Communicates failures through typed, actionable errors

## License

[MIT](LICENSE)
