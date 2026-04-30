# Git Repo Classroom Implementation Plan

Goal: add a Git repository input mode to OpenMAIC so users can submit a GitHub repo URL or trusted local/archive source and generate an interactive codebase tutorial classroom.

Architecture: add a repository-understanding layer before the existing classroom generation pipeline. The layer safely ingests a repo, scans metadata/source structure, samples key files, summarizes the codebase into a compact `RepoKnowledgePack`, plans a developer-oriented curriculum, then reuses existing scene/content/action generation and classroom persistence.

Tech stack: Next.js App Router, TypeScript, Vitest, Node `child_process`/`fs`, existing OpenMAIC `callLLM`, prompt template system, `generateClassroom`, Zustand/React UI.

## Non-goals for MVP

- No private repository authentication.
- No arbitrary command execution inside the target repo.
- No full semantic code index or embeddings.
- No automatic video rendering/export.
- No support for every VCS host. MVP supports GitHub HTTPS URLs and existing local directories on trusted self-hosted deployments.
- No new scene type. Use existing `slide`, `interactive`, `quiz`, and `pbl` scene types.

## Security requirements

- Do not run repository code during analysis.
- Clone with `--depth 1` and optional `--branch` only.
- Reject non-HTTP(S), non-GitHub remote URLs in hosted mode.
- Apply existing SSRF checks to user-provided URLs before clone/fetch.
- Skip sensitive files: `.env*`, `*.pem`, `*.key`, `id_rsa*`, `.npmrc`, `.pypirc`, `credentials`, secret directories.
- Skip heavy/generated directories: `.git`, `node_modules`, `.next`, `dist`, `build`, `coverage`, `target`, `vendor`, `.venv`, `__pycache__`.
- Limit repo clone/extraction size, max file count, max single file bytes, and total sampled bytes.
- Redact secret-looking strings before sending code snippets to LLM.
- Store cloned repos under `data/repo-workspaces/` or a server temp dir that is not publicly served.

## User flow

1. User opens home page.
2. User selects source mode: `Text/PDF` or `Git Repo`.
3. Git Repo mode asks for GitHub URL, optional branch/tag, teaching goal, target audience, depth (`overview | intermediate | deep-dive`), and lesson count.
4. User clicks generate.
5. Backend creates a repo classroom job.
6. Frontend polls job state.
7. On success, user is sent to `/classroom/[id]`.

## Data model

Create `lib/repo/types.ts` with:

- `RepoSourceKind = 'github' | 'local' | 'archive'`
- `RepoSourceInput`
- `RepoLessonDepth = 'overview' | 'intermediate' | 'deep-dive'`
- `RepoClassroomInput`
- `RepoScanLimits`
- `RepoFileSummary`
- `RepoDependencySummary`
- `RepoCodeSample`
- `RepoScanResult`
- `RepoModuleSummary`
- `RepoFlowSummary`
- `RepoKnowledgePack`
- `RepoCurriculumPlan`

Keep field names aligned with the implementation issue in GitHub.

## Implementation tasks

### Task 1: repo type definitions and constants

Files:
- `lib/repo/types.ts`
- `lib/repo/constants.ts`
- `tests/repo/repo-types.test.ts`

Constants:
- `DEFAULT_REPO_SCAN_LIMITS = { maxFiles: 5000, maxFileBytes: 512 * 1024, maxTotalSampleBytes: 400_000, maxTreeEntries: 1200 }`
- `SKIPPED_REPO_DIRS` includes `.git`, `node_modules`, `.next`, `dist`, `build`, `coverage`, `target`, `vendor`, `.venv`, `venv`, `__pycache__`, `.pytest_cache`, `.turbo`.
- `SENSITIVE_REPO_FILE_PATTERNS` covers `.env*`, `.pem`, `.key`, `id_rsa*`, `id_ed25519*`, `.npmrc`, `.pypirc`, `credentials`, `secret`.
- `TEXT_FILE_EXTENSIONS` covers common source/docs/config extensions.

### Task 2: safe repo file filtering and secret redaction

Files:
- `lib/repo/file-filter.ts`
- `lib/repo/redact.ts`
- `tests/repo/file-filter.test.ts`
- `tests/repo/redact.test.ts`

Implement:
- `normalizeRepoPath`
- `shouldSkipRepoPath`
- `isTextLikeRepoFile`
- `detectLanguageFromPath`
- `redactSecrets`

### Task 3: dependency and command detection

Files:
- `lib/repo/dependency-detector.ts`
- `tests/repo/dependency-detector.test.ts`

Implement deterministic manifest heuristics:
- Node: parse `package.json`, detect pnpm/yarn/npm from lockfiles, infer install/dev/test/build commands.
- Python: `pyproject.toml` or `requirements.txt`.
- Go: `go.mod`.
- Rust: `Cargo.toml`.
- Java: `pom.xml`.
- .NET: `.csproj`.

### Task 4: local repository scanner

Files:
- `lib/repo/repo-scanner.ts`
- `tests/repo/repo-scanner.test.ts`

Implement `scanLocalRepo(source, options)`:
- Recursive walk without following symlinks.
- Apply skip/text filters.
- Respect scan limits.
- Build capped `treeText`.
- Build file summaries and roles.
- Read manifests for dependency detector.
- Pick samples in priority order: README/docs, manifests, entrypoints, API routes, central source files, tests.
- Redact sample contents.

### Task 5: safe GitHub clone support

Files:
- `lib/repo/github-source.ts`
- `tests/repo/github-source.test.ts`

Implement:
- `parseGitHubRepoUrl(url)` — HTTPS only, hostname exactly `github.com`, path owner/repo.
- `buildSafeGitCloneArgs(url, dest, branch?)` returns `['clone', '--depth', '1', ...]`.
- `cloneGitHubRepo(source, workspaceRoot?)` with `child_process.spawn` or `execFile`, no shell interpolation.
- Use existing SSRF URL validation if available.
- Workspace path under `data/repo-workspaces/<owner>-<repo>-<timestamp>`.

### Task 6: repo prompt templates and prompt IDs

Files:
- `lib/prompts/types.ts`
- `lib/prompts/index.ts`
- `lib/prompts/templates/repo-summary/system.md`
- `lib/prompts/templates/repo-summary/user.md`
- `lib/prompts/templates/repo-curriculum/system.md`
- `lib/prompts/templates/repo-curriculum/user.md`
- `tests/prompts/repo-prompts.test.ts`

Prompt rules:
- `repo-summary` returns only valid JSON matching `RepoKnowledgePack`.
- `repo-curriculum` returns only valid JSON matching `RepoCurriculumPlan`.
- Do not invent files, APIs, dependencies, or behavior not present in scan.
- Cite concrete source file paths for architecture claims.

### Task 7: repo summarizer and curriculum planner

Files:
- `lib/repo/repo-summarizer.ts`
- `lib/repo/curriculum-planner.ts`
- `tests/repo/repo-summarizer.test.ts`
- `tests/repo/curriculum-planner.test.ts`

Implement:
- Use existing `buildPrompt` and JSON repair/parsing infrastructure.
- Accept `AICallFn` dependency rather than importing server-only model resolution.
- Clamp lesson count to 3-12.
- Provide meaningful error on invalid JSON.

### Task 8: repo classroom generation service

Files:
- `lib/server/repo-classroom-generation.ts`
- `tests/server/repo-classroom-generation.test.ts`

Implement:
- `buildRepoClassroomRequirement(pack, curriculum, input)` producing a bounded text requirement.
- `generateRepoClassroom(input, options)` that scans, summarizes, plans, then calls existing `generateClassroom`.
- Forward media/TTS/agent flags.
- Do not pass raw full source to `generateClassroom`; pass compact repo context and curriculum.

### Task 9: async job store and API routes

Files:
- `lib/server/repo-classroom-job-store.ts`
- `lib/server/repo-classroom-job-runner.ts`
- `app/api/generate-repo-classroom/route.ts`
- `app/api/generate-repo-classroom/[jobId]/route.ts`
- `tests/server/repo-classroom-api.test.ts`

Mirror existing generate-classroom job flow. POST body shape:

```json
{
  "source": { "kind": "github", "url": "https://github.com/THU-MAIC/OpenMAIC", "branch": "main" },
  "teachingGoal": "Teach this repo to intermediate TypeScript developers",
  "targetAudience": "Intermediate web developers",
  "depth": "intermediate",
  "lessonCount": 8,
  "enableWebSearch": false,
  "enableImageGeneration": false,
  "enableVideoGeneration": false,
  "enableTTS": false,
  "agentMode": "default"
}
```

### Task 10: Git Repo mode home UI

Files:
- `components/generation/repo-input-panel.tsx`
- `lib/repo/client-request.ts`
- `app/page.tsx`
- `tests/repo/client-request.test.ts`

Requirements:
- Existing Text/PDF mode remains default and unchanged.
- Add source mode toggle.
- Git Repo fields: GitHub URL, branch/tag, teaching goal, audience, depth select, lesson count.
- POST `/api/generate-repo-classroom` and poll job state.

### Task 11: i18n strings

Add repo mode strings to locale dictionaries and pass:

```bash
corepack pnpm check:i18n-keys
```

### Task 12: docs and examples

Files:
- `README.md`
- `README-zh.md`
- `docs/repo-classroom.md`

Include feature overview, limitations, security notes, request body, course example, and local dev commands.

### Task 13: verification

Run:

```bash
corepack pnpm vitest run tests/repo tests/server/repo-classroom-api.test.ts tests/prompts/repo-prompts.test.ts
corepack pnpm test
corepack pnpm build
```

Manual smoke:
- Submit a small public GitHub repo.
- Confirm scan -> summarize -> curriculum -> classroom generation.
- Result lessons cite real source file paths.
- No skipped/secrets files appear.

## Recommended Codex prompt

```text
Implement the Git Repo Classroom feature following docs/plans/2026-04-26-git-repo-classroom.md.
Work task-by-task in order. Use strict TDD: write failing tests first, run them, implement minimal code, run tests again. Commit after each task. Do not skip security requirements. Do not run code from analyzed target repositories. Do not read or send .env/secrets to LLM prompts. After each task, report changed files and test commands. Final validation: corepack pnpm test && corepack pnpm build.
```

## Codex command

```bash
codex exec --full-auto 'Implement the Git Repo Classroom feature following docs/plans/2026-04-26-git-repo-classroom.md. Work task-by-task in order using strict TDD. Commit after each task. Do not skip security requirements. Do not execute target repo code. Do not read or send .env/secrets to LLM prompts. Final validation: corepack pnpm test && corepack pnpm build.'
```
