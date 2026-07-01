# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository shape

This is the **ONLYOFFICE Docs / DocumentServer** superproject. The top-level repo contains almost no code itself — it aggregates independent projects as git submodules:

| Path | What it is |
|---|---|
| `core` | C++ engine: format converters (OOXML, ODF, PDF, DjVu, XPS, HTML, RTF, TXT, Epub, Fb2, Hwp, MsBinary, OFD...) and the `x2t` conversion tool used by the server. |
| `sdkjs` | JavaScript SDK implementing document editing logic (`word`, `cell`, `slide`, `pdf`, `visio`, `common`) — the "business logic" behind the editors. |
| `web-apps` | Frontend UI (`apps/documenteditor`, `apps/spreadsheeteditor`, `apps/presentationeditor`, `apps/pdfeditor`, `apps/common`, ...), Backbone-based. |
| `server` | Node.js backend: `DocService` (co-editing/collaboration + WOPI), `FileConverter` (drives `core`'s `x2t`), `Common` (shared config/utils/storage/queues), `Metrics`. |
| `core-fonts`, `dictionaries`, `document-templates`, `document-formats` | Static data submodules (fonts, Hunspell dictionaries, sample templates, format definitions). No build system of their own. |
| `sdkjs-plugins/*` | Individual editor plugin submodules (macros, ocr, photoeditor, speech, thesaurus, translator, youtube, zotero, mendeley, highlightcode). |

Because each submodule is checked out at a pinned commit, **always check which submodule a file lives in before editing** — most real work happens inside `server`, `sdkjs`, or `web-apps`, not at the top level. The top-level repo's own history is mostly submodule-pointer bumps and changelog/version updates.

`myrepos` (`mr`) drives multi-repo operations; see `.mrconfig` for canonical clone URLs and `mr-update.sh` for the pull-all-branches helper.

## Fork-specific submodule remotes

`sdkjs` and `web-apps` point at forks (`furtherref/onlyoffice-documentserver-sdkjs`, `furtherref/onlyoffice-documentserver-webapps`) rather than upstream ONLYOFFICE, and each has both remotes configured:

```
sdkjs:     origin -> furtherref/onlyoffice-documentserver-sdkjs.git   upstream -> ONLYOFFICE/sdkjs.git
web-apps:  origin -> furtherref/onlyoffice-documentserver-webapps.git  upstream -> ONLYOFFICE/web-apps.git
```

Fork-only fields/behavior (e.g. plugin `panelWidth`) exist outside upstream's documented API — call these out in code/comments as fork-specific so they aren't confused with upstream ONLYOFFICE plugin API.

**Workflow for changes that span `sdkjs`/`web-apps`:**
1. Create a matching feature branch inside each affected submodule: `git -C sdkjs switch -c <branch>`, `git -C web-apps switch -c <branch>`.
2. Implement and commit inside each submodule normally (`git -C <submodule> commit ...`).
3. Build/verify the submodule (see build commands below).
4. Push each submodule branch to its `origin` (fork), not `upstream`.
5. Only then stage the updated submodule pointers at the top level: `git add sdkjs web-apps .gitmodules` and commit in the parent repo.

Past implementation plans for this kind of cross-submodule change are recorded under `docs/superpowers/plans/<year>/<month>/<day>/*.md` and are useful precedent for how this fork structures multi-submodule work.

## Server (`server/`) — Node.js backend

Each subdirectory (`Common`, `DocService`, `FileConverter`, `Metrics`) is its own npm package with its own `package.json`/`npm-shrinkwrap.json`; the root `server/package.json` only orchestrates them.

```bash
cd server
npm run install:Common          # or install:DocService / install:FileConverter / install:Metrics
npm run build                   # installs all four in parallel

npm run lint:check              # eslint (flat config in eslint.config.js)
npm run lint:fix
npm run format:check            # prettier
npm run format:fix
npm run code:check               # lint:check + format:check
npm run code:fix

npm run tests                   # full jest suite (server/tests, DocService as cwd)
npm run "unit tests"             # unit only
npm run "integration tests with server instance"
npm run storage-tests
npm run "integration database tests"     # passes with no tests if suite absent
npm run "integration rabbitmq tests"     # passes with no tests if suite absent
```

Jest config lives at `server/tests/jest.config.js` (setup file `server/tests/env-setup.js`); tests actually run with `DocService/` as the working directory (see the `cd ./DocService &&` prefix in each script) and match `**/?(*.)+(spec|tests).[tj]s?(x)`. To run a single test file directly:

```bash
cd server/DocService && npx jest ../tests/unit/utils.tests.js --inject-globals=false --config=../tests/jest.config.js
```

`server/tests/fixtures/info/` holds sample `info.json` fixtures consumed by `DocService/sources/routes/info.js` when `USE_FIXTURES = true` (see `server/tests/fixtures/README.md`).

Architecture inside `server`:
- `Common/sources` — shared config loading (`server/Common/config/*.json` per environment), storage backends (`storage/`), queue drivers (`taskqueueMemory.js` / `taskqueueRabbitMQ.js`), pub/sub (`pubsubMemory.js`/`pubsubRabbitMQ.js` live in DocService but mirror this split), logging, licensing.
- `DocService/sources` — the co-editing/collaboration server itself: `DocsCoServer.js` is the main entry logic, `converterservice.js`/`taskresult.js` drive conversion job tracking, `wopiClient.js`/`wopiUtils.js` implement WOPI, `routes/` holds HTTP endpoints.
- `FileConverter/sources` — `convertermaster.js`/`converter.js` shell out to `core`'s compiled `x2t` binary (paths resolved via `converterPaths.js`) to perform actual format conversion.
- `Metrics` — statsd/metrics reporting.

The production build (Makefile-driven, invoked via `grunt` per `Gruntfile.js`) stamps `Common/sources/commondefines.js` and `Common/sources/license.js` with version/build-date info and copies in `core-fonts`, `document-templates`, and license files — this is the packaging path, not something you typically need to run for source changes.

## sdkjs and web-apps (frontend/editor logic)

Both are Grunt-based and normally built together:

```bash
cd web-apps/build
npx grunt deploy-documenteditor        # or deploy-spreadsheeteditor / deploy-presentationeditor / deploy-pdfeditor
```

```bash
cd sdkjs/build && npx grunt            # builds word/sdk-all.js etc.
```

`sdkjs/Makefile` ties the two together (builds sdkjs, then invokes the matching `deploy-<editor>-component` grunt task in `web-apps/build`, then copies both into `deploy/`) — use it when you need a full combined build rather than a single editor.

`web-apps/test/` holds editor-specific browser test fixtures (`documenteditor/`, `spreadsheeteditor/`, `presentationeditor/`, `common/`, `unit-tests/`). `sdkjs` has format-specific tests under e.g. `sdkjs/pdf/test`.

Because right-panel/plugin UI (`RightMenu.js`, `Viewport.js`) is duplicated per editor app rather than shared, changes to that behavior typically need repeating across `apps/documenteditor`, `apps/spreadsheeteditor`, `apps/presentationeditor`, and `apps/pdfeditor` — check all four before assuming a fix is complete.

## core (C++ engine)

`core` is a large native C++ codebase (converters, `X2tConverter`, `DesktopEditor`, etc.) built via the Python/CMake tooling under `core/build`. It's out of scope for most agentic sessions unless a task explicitly touches native conversion behavior — prefer investigating `server/FileConverter` first, since that's the JS-facing boundary that shells out to the compiled `x2t` binary.

## CI

`.github/workflows/check.yml` only lints/spellchecks `CHANGELOG.md` and `ROADMAP.md` at the top level (markdownlint + aspell with `.aspell.en.pws` as the personal dictionary). Submodule CI (server tests/lint, web-apps/sdkjs builds) lives in each submodule's own repo, not here.
