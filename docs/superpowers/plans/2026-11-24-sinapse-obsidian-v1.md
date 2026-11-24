# Sinapse for Obsidian v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A desktop-only Obsidian plugin that edits the vault's `sinapse.json`, runs the `sinapse` executable to reindex and open the savings report, and shows the `claude mcp add` command.

**Architecture:** Pure modules (`sinapseConfig`, `sinapseCli`, `executable`, `version`) with injected fs/process functions, unit-tested with vitest and free of `obsidian` imports. A thin UI shell (`main.ts`, `settings.ts`, `folderSuggest.ts`) wires them to Obsidian APIs and is validated manually.

**Tech Stack:** TypeScript, Obsidian API (`obsidian` npm, latest 1.14.x), esbuild, vitest, Node `child_process.execFile`, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-11-24-sinapse-obsidian-design.md` — read it first.

## Global Constraints

- Repo root: the repository root of `https://github.com/VoidPep/Sinapse-Obsidian.git`. Branches: `main` (releases), `develop` (integration), work branch `feat/plugin-v1` created from `develop`. Do not push.
- Commit messages: conventional style, **no** `Co-Authored-By` or "Generated with" trailers.
- `manifest.json` (already created): id `sinapse`, `isDesktopOnly: true`, `minAppVersion` `1.5.0`. Keep it at repo root.
- Minimum Sinapse executable version: `0.1.0`.
- Executable is never downloaded by the plugin.
- `execFile` only (never a shell), `cwd` = vault root, timeout 600000 ms, one run at a time.
- All paths written to `sinapse.json` are vault-relative with `/`.
- Pure modules must not import `obsidian`.
- No reference to any company, username or local machine path in code, tests or docs.
- UI strings in Portuguese.

## Review Focus

1. **Existing `sinapse.json` with extra/unknown fields or hand edits** — plugin must preserve them byte-for-meaning; only `folders`, `exclude`, `dataDir` change. → Task 2 tests.
2. **`sinapse.json` with invalid JSON** — never overwritten silently; "recreate" backs up to `sinapse.json.bak`. → Task 2 tests.
3. **Executable missing, not executable (Linux, no `chmod +x`), or older than 0.1.0** — clear message, no crash, no unhandled promise. → Tasks 3–4 tests.
4. **Exe prints non-JSON (crash, old version without `--json`) or exits 2** — readable error from stderr / "Configuração inválida". → Task 4 tests.
5. **Clicking "Reindexar" twice, or palette + settings at once** — second call returns "já em andamento" without spawning. → Task 4 test.

---

### Task 1: Toolchain

**Files:**
- Create: `package.json`, `tsconfig.json`, `esbuild.config.mjs`, `version-bump.mjs`, `vitest.config.ts`, `src/main.ts` (minimal `Plugin` subclass with empty `onload`), `.github/workflows/ci.yml`
- Test: `tests/smoke.test.ts`

**Interfaces:**
- Produces: scripts `npm run dev` (esbuild watch), `npm run build` (`tsc -noEmit -skipLibCheck && node esbuild.config.mjs production`), `npm test` (vitest run), `npm run lint`, `npm run version` (`node version-bump.mjs && git add manifest.json versions.json`). Bundle output `main.js` at repo root, format `cjs`, externals `obsidian`, `electron`, `@codemirror/*`, Node builtins.

- [ ] **Step 1:** `git switch -c feat/plugin-v1` from `develop`.
- [ ] **Step 2:** Create the files following the official obsidian-sample-plugin conventions; devDependencies pinned to current stable (`obsidian`, `typescript`, `esbuild`, `builtin-modules` or `node:module` builtins, `vitest`, `@types/node`, eslint + `eslint-plugin-obsidianmd` if it installs cleanly).
- [ ] **Step 3: Write test** `tests/smoke.test.ts`: `expect(1 + 1).toBe(2)` (proves vitest wiring; delete in Task 2).
- [ ] **Step 4:** `npm install && npm run build && npm test && npm run lint` → all pass, `main.js` produced (ignored by git).
- [ ] **Step 5:** `ci.yml`: on push/PR to `main` and `develop`; `ubuntu-latest` and `windows-latest`; Node 22 LTS; `npm ci`, `npm run lint`, `npm test`, `npm run build`.
- [ ] **Step 6: Commit** `chore: toolchain, build and ci`.

---

### Task 2: `sinapseConfig` (read / merge / write)

**Files:**
- Create: `src/sinapseConfig.ts`
- Test: `tests/sinapseConfig.test.ts`

**Interfaces:**
- Produces:
  - `export interface SinapseJson { name?: string; base?: string; folders?: string[]; include?: string[]; exclude?: string[]; dataDir?: string; [k: string]: unknown }`
  - `export interface FsLike { exists(p: string): Promise<boolean>; read(p: string): Promise<string>; write(p: string, data: string): Promise<void>; rename(from: string, to: string): Promise<void> }`
  - `export type LoadResult = { kind: "ok"; json: SinapseJson } | { kind: "missing" } | { kind: "invalid"; error: string }`
  - `export function defaults(vaultName: string): SinapseJson` — spec defaults.
  - `export async function load(fs: FsLike, path: string): Promise<LoadResult>`
  - `export function merge(current: SinapseJson, patch: { folders?: string[]; exclude?: string[]; dataDir?: string }): SinapseJson` — only those keys change; folder entries normalized `\`→`/`, trimmed, no leading `./` or trailing `/`, de-duplicated, empty strings dropped.
  - `export async function save(fs: FsLike, path: string, vaultName: string, patch: …): Promise<LoadResult>` — missing → write `merge(defaults, patch)`; ok → write `merge(json, patch)`; invalid → return the invalid result **without writing**. JSON written with 2-space indent + trailing newline.
  - `export async function recreate(fs: FsLike, path: string, vaultName: string): Promise<void>` — rename existing to `<path>.bak` (overwrite older .bak), then write defaults.

- [ ] **Step 1: Write failing tests:**
  - `save on missing file writes defaults with name = vault name and the patch`.
  - `save preserves unknown keys and include` — current `{ base:".", include:["x/*.md"], custom:{a:1}, folders:["a"] }` + patch `{ folders:["b"] }` → written JSON has `include:["x/*.md"]`, `custom:{a:1}`, `folders:["b"]`.
  - `save on invalid JSON does not write` — `fs.write` never called; result `kind:"invalid"`.
  - `merge normalizes folders` — `["a\\b", "./c/", "a/b", "  "]` → `["a/b", "c"]`.
  - `recreate backs up then writes defaults` — rename called with `sinapse.json` → `sinapse.json.bak` before write.
  - `load distinguishes missing / invalid / ok`.
- [ ] **Step 2:** `npm test` → FAIL.
- [ ] **Step 3:** Implement per Interfaces (in-memory `FsLike` fake in the test file).
- [ ] **Step 4:** `npm test` → PASS. Delete `tests/smoke.test.ts`.
- [ ] **Step 5: Commit** `feat: sinapse.json load/merge/save`.

---

### Task 3: `executable` + `version`

**Files:**
- Create: `src/executable.ts`, `src/version.ts`
- Test: `tests/executable.test.ts`, `tests/version.test.ts`

**Interfaces:**
- Produces:
  - `export function compareVersions(a: string, b: string): number` — semver `X.Y.Z` (ignore pre-release/build suffix); negative/0/positive.
  - `export const MIN_VERSION = "0.1.0"`
  - `export function candidateNames(platform: NodeJS.Platform): string[]` — `win32` → `["sinapse-win-x64.exe", "sinapse.exe"]`; `linux` → `["sinapse-linux-x64", "sinapse"]`; others → `[]`.
  - `export async function resolveExecutable(opts: { vaultPath: string; configured: string; platform: NodeJS.Platform; exists: (p: string) => Promise<boolean>; isAbsolute: (p: string) => boolean; join: (...p: string[]) => string }): Promise<{ path: string } | { error: string }>` — configured non-empty → absolute as-is, relative joined to vault; must exist. Empty → first existing candidate in vault root. Errors (Portuguese): not found (list names searched), unsupported platform ("Sem executável do Sinapse para esta plataforma").

- [ ] **Step 1: Write failing tests:** `compareVersions("0.1.0","0.1.0")===0`, `("0.2.0","0.10.0")<0`, `("1.0.0-beta","1.0.0")===0`; `candidateNames("darwin")` empty; resolve prefers configured relative path joined to vault; falls back to first existing candidate; error message lists searched names when none exist; darwin → unsupported error.
- [ ] **Step 2:** `npm test` → FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** `npm test` → PASS.
- [ ] **Step 5: Commit** `feat: executable discovery and version compare`.

---

### Task 4: `sinapseCli` (run + interpret)

**Files:**
- Create: `src/sinapseCli.ts`
- Test: `tests/sinapseCli.test.ts`

**Interfaces:**
- Consumes: `compareVersions`, `MIN_VERSION`.
- Produces:
  - `export type ExecFn = (file: string, args: string[], opts: { cwd: string; timeout: number }) => Promise<{ code: number; stdout: string; stderr: string }>` (real impl wraps `child_process.execFile`; ENOENT/EACCES mapped to `code: -1` and `stderr` = the error message).
  - `export type CliResult<T> = { ok: true; data: T } | { ok: false; code: number; error: string }`
  - `export class SinapseCli { constructor(exec: ExecFn, exe: string, configPath: string, vaultPath: string); version(): Promise<CliResult<{ version: string; supported: boolean }>>; index(): Promise<CliResult<{ files: number; chunks: number; ms: number }>>; report(): Promise<CliResult<{ path: string }>>; get busy(): boolean }`
  - Args: `version` → `["--version","--json"]`; `index` → `["index","--config",configPath,"--json"]`; `report` → `["report","--config",configPath,"--json"]`. Timeout 600000. `cwd` = vaultPath.
  - Interpretation: last stdout line that parses as JSON object; `ok:true` → data; `ok:false` → its `error`; none → `stderr.trim()` or `"Saída inesperada do sinapse (código N)"`. Exit `2` → error prefixed `"Configuração inválida: "`. `code -1` with EACCES → append `" — no Linux, rode: chmod +x <exe>"`.
  - Concurrency: while an `index` or `report` is running, another `index`/`report` returns `{ ok:false, code:-2, error:"Já existe uma execução do Sinapse em andamento." }` without calling exec.

- [ ] **Step 1: Write failing tests** (fake `ExecFn`):
  - `index parses ok json` — stdout `"warn\n{\"ok\":true,\"files\":3,\"chunks\":9,\"ms\":12}\n"` → data `{files:3,chunks:9,ms:12}`.
  - `index passes config and json flags` — captured args equal `["index","--config","/v/sinapse.json","--json"]`, cwd `/v`, timeout 600000.
  - `exit 2 maps to configuração inválida` — stdout `{"ok":false,"error":"Pasta base não encontrada: x"}`, code 2 → error `"Configuração inválida: Pasta base não encontrada: x"`.
  - `no json falls back to stderr`.
  - `EACCES suggests chmod`.
  - `version below minimum is unsupported` — `{"ok":true,"version":"0.0.9"}` → `supported:false`.
  - `second call while busy is rejected without exec` — exec called once.
- [ ] **Step 2:** `npm test` → FAIL.
- [ ] **Step 3:** Implement (plus `nodeExec: ExecFn` using `child_process.execFile`, `windowsHide: true`, `maxBuffer` 10 MB).
- [ ] **Step 4:** `npm test` → PASS.
- [ ] **Step 5: Commit** `feat: sinapse cli runner`.

---

### Task 5: Plugin shell (settings tab, folder picker, commands)

**Files:**
- Modify: `src/main.ts`
- Create: `src/settings.ts`, `src/folderSuggest.ts`, `styles.css` (minimal)

**Interfaces:**
- Consumes: everything above.
- Produces: `interface PluginSettings { executablePath: string; configPath: string; reindexOnStartup: boolean }` defaults `{ executablePath: "", configPath: "sinapse.json", reindexOnStartup: false }`, persisted with `loadData/saveData`.

- [ ] **Step 1:** `main.ts`: on load, read settings; build `FsLike` over Node `fs/promises` with absolute paths from `(app.vault.adapter as FileSystemAdapter).getBasePath()` (if adapter isn't `FileSystemAdapter`, show Notice and disable); register commands `sinapse-reindex` ("Reindexar") and `sinapse-open-report` ("Abrir relatório de economia"); if `reindexOnStartup`, run index in `app.workspace.onLayoutReady` without blocking.
- [ ] **Step 2:** `settings.ts`: sections exactly as spec "Aba de configurações" 1–8, using `Setting`. Field changes in 2–4 call `save` with ~500 ms debounce; `kind:"invalid"` → render error + "Recriar com padrões" button (`recreate`). Reindex button disabled while `cli.busy`; status text updated with result. Report button → `report()` then open the returned path (`electron` `shell.openPath` via `require('electron')`, fallback `window.open('file://'+path)`). Claude Code section: read-only text with absolute exe and config paths quoted, plus "Copiar" (`navigator.clipboard.writeText`).
- [ ] **Step 3:** `folderSuggest.ts`: `FuzzySuggestModal<TFolder>` listing `app.vault.getAllLoadedFiles()` folders except root, `.obsidian` and the data dir; on choose → add vault-relative path.
- [ ] **Step 4:** `npm run build && npm test && npm run lint` → pass.
- [ ] **Step 5: Manual check** in a throwaway test vault (copy `main.js`, `manifest.json`, `styles.css` into `<vault>/.obsidian/plugins/sinapse/`, put the Sinapse v0.1.0 exe in the vault root): Testar → shows version; add 2 folders → `sinapse.json` updated, unknown key added by hand survives; Reindexar → status shows counts; break the JSON by hand → error + recreate creates `.bak`; palette commands work; report opens in the browser; copied `claude mcp add` command runs. Record results in the task report (the main agent decides whether to ask the user for the Fedora run).
- [ ] **Step 6: Commit** `feat: settings tab, folder picker and commands`.

---

### Task 6: Release + README

**Files:**
- Create: `.github/workflows/release.yml`, `README.md` (replace stub), `CHANGELOG.md`

- [ ] **Step 1:** `release.yml`: on tag matching `[0-9]+.[0-9]+.[0-9]+`; Node 22; `npm ci && npm run build`; fail if tag ≠ `manifest.json` version; create GitHub release (draft) attaching `main.js`, `manifest.json`, `styles.css` (`softprops/action-gh-release`, pinned major).
- [ ] **Step 2:** `README.md` (Portuguese, generic): what it does; requirement (Sinapse exe ≥ 0.1.0 from the Sinapse-RAG releases, placed in vault root; `chmod +x` on Linux); install via BRAT (repo `VoidPep/Sinapse-Obsidian`); each setting; commands; how to connect Claude Code; desktop only; release process (`npm version X.Y.Z` → push tag on `main`). `CHANGELOG.md`: `## 0.1.0`.
- [ ] **Step 3:** `npm run build && npm test && npm run lint` → pass.
- [ ] **Step 4: Commit** `docs: readme, changelog and release workflow`.

---

## Out of this plan

- Main agent: push `feat/plugin-v1` and open PR → `develop`.
- User: Fedora manual run, merge to `main`, `npm version 0.1.0` tag, publish the draft release, later submission to the community store.
