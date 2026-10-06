# Miro connector research

**Decision:** implement Miro on the Go syncer path, `internal/syncer/connector`, and register it from `RegisterBuiltIns`. That is the path the current syncer guide requires for a new source, and it is the process that runs when `API_PROXY_SCHEME=go`.

A pull request is acceptable when it does one thing, includes tests for the new behavior, describes the change in the PR template, and passes the CI that actually runs on a non-draft PR labeled `ci`. Constructor config field names must match the fields the frontend saves. The connector reads the external source and emits documents. It does not write the RAGFlow database.

`common/data_source` is still on the runtime path for `python` and `hybrid`. It is not a leftover. A Go-only Miro connector will sync and test-connect only on the Go scheme. Whether python/hybrid must also get a Miro implementation is an open product question, recorded at the end. The syncer guide does not list a Python file in "Adding a new connector".

## 1. What a PR must satisfy

### Contribution guide

Source: `docs/develop/contributing.md`, heading "Contribution Guidelines".

- Fork, branch, commit with a message that explains the change, push, and open a PR ("File a Pull Request (PR)" → "General Workflow").
- Split a large change into smaller standalone PRs. Address one issue. Add test cases for a new feature ("Before Filing a PR").
- Title is concise. Refer to a GitHub issue when there is one. Breaking changes and API changes need design detail in the description ("Describing Your PR").
- The PR passes all CI before merge ("Reviewing & Merging a PR"). The guide does not name the workflows.

### PR template

Source: `.github/pull_request_template.md`.

The template has one section, **Summary**: briefly describe what the PR solves, with background for reviewers. It has no checklist, test plan, or label requirement.

### Feature-request issue template

Source: `.github/ISSUE_TEMPLATE/feature_request.yml`. This is an issue form, not a PR gate.

Required before the issue is accepted for response:

- Searched existing issues, including closed ones.
- Report is in English, and the title is English (language policy linked from the form).
- Template is filled and not modified.
- "Describe the feature you'd like" is required.

Optional and useful as design context: the problem, implementation considered, documentation and use case, additional information.

### Repo working rules that govern this change

Source: `AGENTS.md`.

**Core Stance.** Legacy code is a liability. Prefer deletion over shims and dual-track notes. If old and new implementations coexist, converge to one path unless an external contract forces compatibility. Remove dead tests and stale docs. Keep the change on the owning abstraction.

**Go-Specific Rules.** `internal/ingestion`, `internal/parser`, and `internal/deepdoc` are the actively refactored trees. Do not add deprecated Go APIs to ease an in-repo migration. Remove commented-out Go code. Doc comments describe the current runtime path.

**Go Test Tiers.** Untagged tests are the unit tier and run by default. They must not need MySQL, MinIO, Elasticsearch, Infinity, or an LLM. Tests that call a real service use `//go:build integration`, `e2e`, or `manual` before the `package` clause. `manual` is local opt-in and is never wired into CI. Verify with `bash build.sh --test`, not bare `go test`.

**Working Rules.** Inspect the owning path before editing. Keep the change small. One implementation path. Preserve still-valid behavior with focused tests. Do not add compatibility wording in comments or docs.

**Validation Preference.** After a Go change, run the narrowest `bash build.sh --test` package first. Frontend changes use the touched package's lint, type-check, or test command. Do not default to raw `go test` or `go build`; they miss the CGO flags and native static libraries that `build.sh` sets.

### CI workflows

Only three workflows exist under `.github/workflows/`.

| Workflow | When it runs | What a connector PR must care about |
| --- | --- | --- |
| `sep-tests` (`.github/workflows/sep-tests.yml`, `name: sep-tests`) | `pull_request` types `synchronize` and `labeled`. Ignores `docs/**`, `*.md`, `*.mdx`. Jobs run only when the PR is not a draft and has the label `ci`. | This is the PR suite. |
| `serenedb` (`.github/workflows/serenedb.yml`) | `workflow_dispatch` only. The file says it never runs on push or pull_request. | Not a PR gate. It runs `./build.sh --test-integration ./internal/engine/serenedb/...`. |
| `release` (`.github/workflows/release.yml`) | Daily schedule and tag pushes `v*.*.*` and `nightly`. | Not a PR gate. It builds the CLI. |

`sep-tests` jobs, from the same file:

- **`ragflow_preflight`.** On a PR, Lefthook `pre-commit` runs check-only on changed files (`LEFTHOOK_CHECK_ONLY=1`), then `lefthook run web-checks`. `lefthook.yml` `pre-commit` jobs that apply to connector files: end-of-file, trailing whitespace, mixed line endings, `ruff check` and `ruff format --check` on `*.py`, `gofmt -l` on `*.go`, plus yaml/json/case/merge/symlink checks. `web-checks` runs oxfmt and oxlint on changed `web/` files and fails if a changed file sits under `web/**/tests/`. If any `*.py` changed, the job runs `uv sync --python 3.13 --group test --frozen` and `python3 run_tests.py -i`.
- **`ragflow_tests_infinity` and `ragflow_tests_elasticsearch`.** Same `ci` / non-draft condition. The matrix scheme comes from changed paths (`detect_changes` in the preflight job). A `*.go` change sets `has_go`. When `API_PROXY_SCHEME` is `go`, the job builds with `./build.sh --cpp` and `./build.sh --go`, then runs unit tests with `./build.sh --test` (or `./build.sh --test -- $PKGS`). `build.sh` `run_go_tests` invokes `go test -tags cgo,static` with no `integration` tag. The package list drops `internal/storage` and `internal/handler`. It also runs `uv run pytest ragflow_deps/test_download_go_deps.py ragflow_deps/test_tokenizer_assets.py`. Later steps build `ragflow:nightly` and run SDK and REST tests against that engine. Those later steps are the full server suite, not connector-specific.
- **`ragflow_go_integration`.** Runs when `has_go_changes` is true, under the same `ci` / non-draft condition. It starts the CI service stack and runs `./build.sh --test-integration-go $PKGS`. `build.sh` maps that flag to `run_go_tests_tagged integration`, which is `go test -tags "integration static"`. The package list excludes `test/api`, `internal/deepdoc`, `internal/storage`, and `internal/service/nlp`. `internal/syncer/connector` is not excluded. Untagged test files are still compiled in that invocation; `//go:build integration` files are included as well. New Miro tests must stay free of a live Miro account. The comment in the job says engine-specific suites self-skip unless their env vars are set.
- **`ragflow_native_backend`.** Runs `./build.sh --test-native` only when `has_native_deepdoc_changes` is true. A connector PR does not enter this job unless it also touches `internal/deepdoc/**`, `internal/binding/cpp/**`, or `internal/deepdoc/native/testdata.ref`.

Local check the syncer guide names (`internal/syncer/README.md`, "Testing"):

```bash
bash build.sh --test ./internal/syncer/connector
bash build.sh --test ./internal/syncer
```

### Frontend rules that apply to registration

Source: `web/AGENTS.md`.

- The data-source screen is shared by the Go and Python backends. Backend differences go through `pickByBackend` / `BackendVariant` from `@/utils/backend-variant`, not through a direct import of `@/utils/backend-runtime`. `web/src/pages/user-setting/data-source/constant/index.tsx` already uses `pickByBackend` for MySQL and PostgreSQL `batch_size` defaults.
- Grouped frontend tests live in `__tests__/`, never `tests/`. Co-located `*.test.ts(x)` is also fine. `constant/index.test.tsx` is the existing catalog test.
- i18n uses `react-i18next`. Add keys only to the language files the task asks for, commonly `web/src/locales/en.ts` and `web/src/locales/zh.ts`. English copy is sentence case. Existing connector descriptions are also present in the other locale files under `web/src/locales/`.
- Do not edit `web/src/components/ui/` to add a field widget. Compose a field outside that folder. Notion does not need a custom widget; Google Drive, Gmail, and Box do (`component/google-drive-token-field.tsx` and siblings).
- A connector form that only extends the existing data-source constants does not add a new request hook. Test connection already goes through the existing connector API.

## 2. Where a connector is implemented

### The path new work follows

Source: `internal/syncer/README.md`, opening note and heading "Adding a new connector".

New data sources plug into `internal/syncer/connector`. The same note says not to add compatibility layers around the old Python sync worker. Constructors stay compatible with config fields the Python side and the frontend already use, when those fields exist. Miro has none yet, so the frontend fields and the Go constructor have to be invented together and then kept identical.

Steps in that section:

1. `internal/syncer/connector/<source>.go`
2. `New<Source>Connector(config map[string]any) (*<Source>Connector, error)` — parse config, no network I/O
3. `Validate`, `OpenSync`, `OpenPrune`
4. `ValidateConnectorSetting` when test connection is supported
5. `Fetcher` only when download is lazy
6. Register in `RegisterBuiltIns` in `internal/syncer/connector/builtin.go`
7. `internal/syncer/connector/<source>_test.go`

`RegisterBuiltIns` (`builtin.go`) registers both the raw-config factory (test connection) and the task-context factory (syncer runtime) for each source string. Notion is `registerBuiltIn(registry, "notion", NewNotionConnector)`.

The Go server starts this registry from `internal/syncer/syncer.go` `NewSyncer`, and the test-connection service uses the same registry from `internal/service/connector.go` `TestConnector` → `OpenFromConfig` → `ValidateConnectorSetting`.

### Python is still executed for python and hybrid

`docker/entrypoint.sh` and `docker/launch_backend_service.sh` start data sync as follows:

- `API_PROXY_SCHEME=go` → `bin/ragflow_server --syncer` (Go syncer, `cmd/ragflow_server.go` `--syncer` mode, `syncer.NewSyncer`).
- Any other scheme, including `hybrid` and `python` → `rag/svr/sync_data_source.py`.

That Python worker imports connectors from `common/data_source` (`rag/svr/sync_data_source.py` imports) and dispatches them through `func_factory` (Notion is `FileSource.NOTION: Notion`). The Python HTTP test endpoint `api/apps/restful_apis/connector_api.py` `test_connector` calls `common.data_source.build_connector_for_source`, which looks up `CONNECTOR_BY_SOURCE` in `common/data_source/__init__.py` and raises `ConnectorValidationError` for an unknown source.

So `common/data_source` is the live sync and test-connection implementation whenever the scheme is not `go`. The Go registry is the live implementation when the scheme is `go`.

### Template: Notion, end to end

Notion is fully wired on both runtimes. Asana is too (`builtin.go` `"asana"`, `common/data_source/asana_connector.py`, `DataSourceKey.ASANA`). Notion is the closer content template: pages become text documents, attachments become separate documents, and prune returns a full id snapshot.

| Piece | Notion file | What it owns |
| --- | --- | --- |
| Go connector | `internal/syncer/connector/notion.go` | `NewNotionConnector`, `Validate`, `ValidateConnectorSetting`, `OpenSync`, `OpenPrune`. Config keys: `credentials.notion_integration_token`, optional `root_page_id`. |
| Go tests | `internal/syncer/connector/notion_test.go` | Untagged unit tests. Remote calls are function hooks (`searchPages`, `fetchPage`, `fetchChildBlocks`), not a live Notion account. |
| Go registry | `internal/syncer/connector/builtin.go` `RegisterBuiltIns` | Source string `"notion"`. There is no Go enum of connector sources. `internal/entity/dataset.go` `FileSource` only defines `""`, `"knowledgebase"`, and `"s3"`, which is a different field. |
| Python connector | `common/data_source/notion_connector.py` | Still imported by the Python worker. |
| Python factory | `common/data_source/__init__.py` `CONNECTOR_BY_SOURCE` | `FileSource.NOTION: NotionConnector`. `build_connector_for_source` is what the Python test endpoint calls. |
| Python source enums | `common/constants.py` `FileSource.NOTION`; `common/data_source/config.py` `DocumentSource.NOTION` | The worker and the factory key off `FileSource`. |
| Python worker | `rag/svr/sync_data_source.py` class `Notion` and `func_factory` | Instantiates `NotionConnector` with `root_page_id` and `credentials`. |
| Frontend catalog | `web/src/pages/user-setting/data-source/constant/index.tsx` | `DataSourceKey.NOTION = 'notion'`. Card copy and icon in the info map. `DataSourceFeatureVisibilityMap` sets `syncDeletedFiles: true`. Form fields `config.credentials.notion_integration_token` (password, required) and `config.root_page_id` (text, optional). Defaults in `DataSourceFormDefaultValues`. |
| Icon | `web/src/assets/svg/data-source/notion.svg` | Referenced as `data-source/notion`. |
| i18n | `web/src/locales/en.ts` `setting.notionDescription`, `dataSourceFieldNotionIntegrationToken`, `dataSourceFieldRootPageId`. The same description key exists in the other files under `web/src/locales/`. | `web/AGENTS.md` says not to fan a new key out to every locale unless asked. |
| Catalog test | `web/src/pages/user-setting/data-source/constant/index.test.tsx` | Existing pattern asserts catalog entry and defaults for a source (the file's first case is Xquik). Notion itself is not asserted there. |
| User guide | `docs/guides/data_source/data_source_categories_and_selection.md` | Names Notion under "Documents and collaboration platforms". There is no per-connector Notion page. `overview_and_page_management.md` describes the shared create/settings flow and does not list sources. |
| Docs CI | `.github/workflows/sep-tests.yml` `paths-ignore` | Changes under `docs/**` and `*.md` do not trigger `sep-tests`. |

Config compatibility, from the README note under "Adding a new connector": the Go constructor reads the same map the frontend saves. Notion's frontend names and `NewNotionConnector` match: `credentials.notion_integration_token` and `root_page_id`.

Notion behavior worth copying, from `notion.go` and `internal/syncer/connector/github.go` helpers `beforeOrAtWindowStart` / `afterWindowEnd`:

- `Validate` checks the token and batch size, then either fetches the root page or searches with `PageSize: 1`. It does not download the workspace.
- `ValidateConnectorSetting` bounds that probe with `connectorSettingValidationTimeout` (`interface.go`, 7 seconds) and then calls `Validate`.
- `OpenSync` keeps the request window fixed. Incremental inclusion is `WindowStart < UpdatedAt <= WindowEnd` (`includeNotionUpdatedAt`).
- Page `SourceID` is the Notion page id. Fingerprint is `stableFingerprint` of id plus `last_edited_time` (`fingerprint.go`).
- Resume stores that id on the checkpoint. If the id is absent from the listing, `OpenSync`/`NextBatch` returns `ErrSyncResumeInvalid`.
- `OpenPrune` walks the full current page set and returns `SlimDocument{SourceID}` for pages and attachments, with no incremental filter.
- The connector does not write the database. The README "IMPORTANT" under "Overall architecture" assigns writes, document ids, checkpoints, and parse scheduling to the syncer.

## 3. Miro API facts

Base URL on the fetched OpenAPI servers: `https://api.miro.com`. Facts below are only from pages fetched for this note.

### Auth

REST apps use the OAuth 2.0 authorization code flow. Web-plugin authorization is for apps that only use the Web SDK.

Sources: [API introduction](https://developers.miro.com/reference/api-reference), [expiring-token flow](https://developers.miro.com/reference/authorization-flow-for-expiring-tokens), [quickstart](https://developers.miro.com/docs/rest-api-build-your-first-hello-world-app).

App setup, from the introduction and the quickstart:

- Create a Developer team, create an app, configure it (manifest or settings UI), install it.
- Token expiry is chosen at app creation and cannot be changed later.
- Expiring access token: 1 hour. Refresh token: 60 days. A refresh issues a new access token and a new refresh token, and invalidates the previous pair.
- Non-expiring access token: valid until the user uninstalls the app from the team.
- The quickstart's service-account note: a non-expiring token from "Install app and get OAuth token" can call the REST API without implementing the browser flow. An expiring token still needs refresh.

Authorize redirect, [step 1](https://developers.miro.com/reference/create-authorization-request-link):

`https://miro.com/oauth/authorize?response_type=code&client_id={client_id}&redirect_uri={redirect_uri}`

Optional query params: `state`, `team_id`. `redirect_uri` must match the URL configured in the app settings. The sample includes a trailing slash (`https://localhost:3000/`).

Token exchange and refresh use the v1 token URL. The refresh page says authentication endpoints stay on v1.

- Exchange: [step 3](https://developers.miro.com/reference/exchange-authorization-code-with-access-token). POST body fields `grant_type=authorization_code`, `client_id`, `client_secret`, `code`, `redirect_uri`. The page says `redirect_uri` must match the original URI, including the trailing slash. Documented responses on that page: 200 and 400.
- Refresh: [step 5](https://developers.miro.com/reference/get-new-access-token-using-refresh-token). `POST https://api.miro.com/v1/oauth/token` with `Content-Type: application/x-www-form-urlencoded` and `grant_type=refresh_token`, `refresh_token`, `client_id`, `client_secret`. The sample JSON response fields are `token_type`, `user_id`, `team_id`, `access_token`, `refresh_token`, `scope`, `expires_in` (example `3599`).

The [authorization model](https://developers.miro.com/reference/authorization-model) shows the same two JSON shapes: expiring responses include `refresh_token` and `expires_in`; non-expiring responses include `access_token`, `token_type`, `scope`, `user_id`, `team_id` and omit `refresh_token`.

### Scopes

Source: [Permission scopes](https://developers.miro.com/reference/scopes).

| Scope | What the page says | Needed for |
| --- | --- | --- |
| `boards:read` | Retrieve information about boards, board members, or items. Web SDK and REST API. | List boards, list items, get one item. The webhook guide also requires it to create a subscription. |
| `boards:write` | Create, update, or delete boards, board members, or items. | Not required to read. |
| `boards:export` | Export boards within the organization as PDF with comments and talktrack. Enterprise. | The enterprise export job. |

`identity:read` is listed as Web SDK only ("Read profile information for current user, including email"). Do not assume it authorizes a REST "who am I" probe.

### Endpoints

| Need | Method and path | Scope and tier | Source |
| --- | --- | --- | --- |
| List boards the token can access | `GET /v2/boards` | `boards:read`, Level 1 | [Get boards](https://developers.miro.com/reference/get-boards-1) |
| List items on a board | `GET /v2/boards/{board_id}/items` | `boards:read`, Level 2 | [Get items](https://developers.miro.com/reference/get-items-1) |
| Get one item | `GET /v2/boards/{board_id}/items/{item_id}` | `boards:read`, Level 1 | [Get specific item](https://developers.miro.com/reference/get-specific-item-1) |
| Export boards to SVG, HTML, or PDF | `POST /v2/orgs/{org_id}/boards/export/jobs` | `boards:export`, Level 4, Enterprise only | [Create board export job](https://developers.miro.com/reference/enterprise-create-board-export) |

List boards query params from that OpenAPI: `team_id`, `project_id` (the page says projects were renamed to Spaces), `query` (max length 500), `owner`, `limit` (default 20, min 1, max 50), `offset` (default 0), `sort` (`default`, `last_modified`, `last_opened`, `last_created`, `alphabetically`). `sort` applies only when searching by team or project. With `team_id`, `query` and `owner` are ignored. With `project_id`, `query` and `owner` are ignored. Passing `team_id` also ignores `owner`.

Board object fields used for sync: `id` ("Unique identifier (ID) of the board", example `uXjVOD6LSME=`), `name`, `description`, `createdAt`, `modifiedAt`, `lastOpenedAt`. Times are described as UTC ISO 8601 with a trailing Z. `modifiedAt` is "Date and time when the board was last modified."

List items query params in the OpenAPI: `limit` (default 10, min 10, max 50), `type`, `cursor`, path `board_id`. The prose says the call can also return child items inside a parent. The parameter list on that page does not include a parent id. A `links.related` example URL contains `parent_item_id`. There is no `modifiedAt` filter on this call.

`type` enum: `text`, `shape`, `sticky_note`, `image`, `document`, `card`, `app_card`, `preview`, `frame`, `embed`, `doc_format`, `data_table_format`. The description says a `document` is an uploaded file such as a PDF, and `doc_format` is a Miro structured document similar to a Google Doc. The example in the same description says set `type` to `cards` for card items; the enum value is `card`.

Item identity and time, from the generic item schema on the list-items page and from [Item](https://developers.miro.com/reference/rest-api-item-model): `id` is required and is the unique item id (example `3458764517517819000`). `modifiedAt` is when the item was last modified. The schema description says ISO 8601 with a trailing Z. The example value on the list-items schema is `2022-03-30 17:26:50+00:00` (space, numeric offset).

Export request body: `boardIds` (1 to 1000) and `boardFormat` enum `SVG` (default), `HTML`, `PDF`. Query `request_id` is a UUID. The page says the caller must be a Company Admin and eDiscovery must be enabled. This is not a general "download my board as PDF" API.

### Pagination, rate limits, errors

Board list pagination is offset-based. Response `BoardsPagedResponse` has `data`, `total`, `size`, `offset`, `limit`, `links`. If `total` is greater than `size`, request again with a higher `offset`. When `query` is set, `offset + limit` must not exceed 10,000. The page says to narrow with `team_id`, `project_id`, or `owner` instead of a higher offset. Filtering by `team_id` or `project_id` is instant; other filters can lag indexing of newly created boards.

Item list pagination is cursor-based. Response `GenericItemCursorPaged` has `data`, `total`, `size`, `cursor`, `limit`, `links`. Pass the returned `cursor` on the next call. The page does not say the cursor is stable across content edits.

Rate limits, [Rate limiting](https://developers.miro.com/reference/rate-limiting). The page says limits can change without notice. Limits are per user per application, measured in credits, with a global cap of 100,000 credits per minute.

| Tier | Cost | Requests per minute at that cost |
| --- | --- | --- |
| Level 1 | 50 credits | 2000 |
| Level 2 | 100 credits | 1000 |
| Level 3 | 500 credits | 200 |
| Level 4 | 2000 credits | 50 |

Response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (UNIX epoch). HTTP 429 body:

```json
{"status": 429, "code": "tooManyRequests", "message": "Request rate limit exceeded", "context": null, "type": "error"}
```

The page's guidance is exponential backoff on 429.

Documented error bodies on `GET /v2/boards` and `GET /v2/boards/{board_id}/items` (same shapes on get-specific-item):

| HTTP | `code` | Description on the page |
| --- | --- | --- |
| 400 | `invalidParameters` | Malformed request |
| 404 | `notFound` | "Invalid access" |
| 429 | `tooManyRequests` | Too many requests |

Those three OpenAPI documents do not list 401 or 403. The enterprise export OpenAPI does: 401 `tokenNotProvided` ("Invalid authentication credentials"), 403 `forbiddenAccess` ("Invalid access"), 404 `notFound` ("Not found"), 429 `tooManyRequests`, plus 400 and 425 `tooEarly`.

Get boards also says an Enterprise Company Admin with Content Admin permissions can list private boards, and that private board contents stay inaccessible. Unauthorized content access "will return an error". The paragraph does not name the status code.

### Webhooks and polling

Webhooks exist for item create, update, and delete on a board.

- [Webhooks using Python](https://developers.miro.com/docs/getting-started-with-webhooks-python): scope `boards:read`; subscription fields `boardId`, `callbackUrl`, `status=enabled`; success is HTTP 201. Event `type` is `create`, `update`, or `delete`. Create and update payloads include `item.id`, `item.type`, and `item.modifiedAt`. Delete payload in the sample is `boardId`, `item.id`, and `type: delete`.
- [Pipedream test endpoint](https://developers.miro.com/docs/set-up-a-test-endpoint-for-webhooks): `POST https://api.miro.com/v2-experimental/webhooks/board_subscriptions` with JSON `status`, `boardId`, `callbackUrl`. Miro POSTs a `challenge` to the callback; the endpoint must answer 200 with the same `challenge`. A successful create returns 201 and a `board_subscription` object.

The subscription is per board, on an experimental path, and it needs a public callback that answers a challenge. The RAGFlow syncer does not ingest webhooks. It wakes tasks on `refresh_freq` (`internal/syncer/README.md`, "Timer re-arm"). Polling `modifiedAt` is the model that fits the syncer. Webhooks are a different ingress.

Enterprise content logs ([Board Content Logs](https://developers.miro.com/reference/board-content-logs)) are an admin audit feed for organizations with logging enabled. They are not the general incremental API.

### Stable ids and content shape

Stable ids the docs define:

- Board `id` on `GET /v2/boards`. Example `uXjVOD6LSME=` includes `=`.
- Item `id` on the item model. Example is a numeric string.
- Board `modifiedAt` and item `modifiedAt` are the update timestamps.

What can become text, from schemas on [Get items](https://developers.miro.com/reference/get-items-1):

| Item `type` | Text the schema exposes | Not text |
| --- | --- | --- |
| `sticky_note` | `data.content` | Shape is visual. |
| `text` | `data.content` (required in `TextData`) | |
| `shape` | `data.content` | |
| `card` | `data.title`, `data.description` | Assignee and due date are metadata. |
| `app_card` | `data.title`, `data.description`, custom field `value` | Status and owned flag are metadata. |
| `embed` | `data.description`, `data.html` | `html` is an embed snippet, not the remote page body. |
| `document` | `data.title` | `data.documentUrl` is a download URL. Default `redirect=false` returns a URL valid for 60 seconds. |
| `image` | `data.title`, `data.altText` | `data.imageUrl` is a download URL, also 60 seconds when `redirect=false`. |
| `frame` | The schema `$ref`s `FrameData` and does not define its properties in the fetched OpenAPI. | |
| `doc_format`, `data_table_format`, `preview` | Named on the `type` enum. Not present in the `WidgetDataOutput` `oneOf` on the fetched page. | |

A board is a container. One board as one markdown document can be built from `name`, `description`, and `modifiedAt` without listing items. Item text requires the Level 2 list (and, for uploaded files, a second download before the 60-second URL expires).

## 4. Design implications for the Go interface

Interface source: `internal/syncer/connector/interface.go` and `models.go`. Semantics source: `internal/syncer/README.md` headings "Connector interfaces", "Data model semantics", "Retry and resume", "Validation", "Testing", "Common pitfalls".

| Interface | Miro mapping for a first slice |
| --- | --- |
| `Validate` | Local check for a non-empty access token, then one lightweight `GET /v2/boards?limit=1`. No item walk. |
| `ValidateConnectorSetting` | Same probe under `connectorSettingValidationTimeout` (7s). Input is unsaved config. No connector id or task id. |
| `OpenSync` | Fixed `WindowStart` / `WindowEnd` from the request. Do not read "now" inside `NextBatch`. |
| Window | Include a board when `WindowStart < modifiedAt <= WindowEnd`. `FromBeginning` ignores the window. Helpers that implement this live in `github.go` (`beforeOrAtWindowStart`, `afterWindowEnd`) and are what `includeNotionUpdatedAt` uses. |
| `SourceID` | The board `id` string exactly as returned. It is unique per the Get boards schema. The example contains `=`, so encoding or trimming the id changes the id. |
| `UpdatedAt` | Board `modifiedAt`. |
| `Fingerprint` | Stable hash of board id plus `modifiedAt`, same idea as Notion's `stableFingerprint` of id plus `last_edited_time`. |
| Checkpoint | After a committed batch, `SyncCheckpoint.SourceID` (and `Cursor`) is the last board id in a stable order. If that id disappears from the listing, return `ErrSyncResumeInvalid`. |
| `OpenPrune` | Full list of current board ids, no time filter, no item bodies. A partial list deletes the rest (`README` caution under "PRUNE pipeline" and "Common pitfalls"). |
| `Fetcher` | Not used in the first slice. Inline markdown goes in `Blob`. |
| DB writes | None. The syncer resolves `kb_id + connector_id + SourceID`, stores checkpoints, and schedules parse. |

Pitfalls from the README that apply here:

- Unstable `SourceID` (normalizing the board id, or using item ids that the API replaces) duplicates documents and makes prune delete the previous ones.
- Unstable batch order breaks resume. `sort=last_modified` is documented only when `team_id` or `project_id` is set. A first slice should collect the page and then emit in a stable order (board id, or `modifiedAt` then id), and checkpoint that order.
- `OpenPrune` is a full snapshot. Returning only boards inside the sync window would delete everything older.
- The connector must not write RAGFlow rows or persist its own cursor.
- `GET /v2/boards` with `query` stops at 10,000 matches. An unfiltered offset walk is the listing that matches "boards this token can access". Whether that unfiltered walk has its own cap is not stated.
- 429 is a transient task error if the error string contains `too many requests` or `http 429` (`internal/syncer/transient_error.go` `isTransientSyncError`). Surface the HTTP status in the error. 429 on test connection should be a clear message, per the README "Validation" section.
- Item download URLs last 60 seconds. They are a bad `FetchRef`: the runner may call `Fetch` later. A later slice that downloads images or PDFs should fetch the item again at download time, or download inside the batch. That is a reason to defer attachments.
- Board `modifiedAt` changes when the board changes. It does not tell you which item changed. One document per board re-ingests the whole board text whenever the board timestamp moves. That is acceptable for the first slice and expensive once item bodies are included.

### Smallest first slice

1. Credential is a pasted access token, stored like Notion's integration token (`config.credentials.…`). The Miro quickstart allows a non-expiring token from app install without a browser flow.
2. `Validate` / test connection: `GET /v2/boards?limit=1` with `boards:read`.
3. Sync: list boards the token can access. One markdown (or `.txt`) document per board, using board `id`, `name`, `description`, and `modifiedAt`.
4. Incremental filter on board `modifiedAt` with the open-closed window. Full-sync flag uses `FromBeginning`.
5. Prune: every current board id, if listing all boards is reliable. If a complete id list is not reliable, return `ErrPruneUnsupported` so prune deletes nothing (README `OpenPrune`).
6. Unit tests with `httptest` or a fetch hook: config parse, missing token, 400/404/429 on validate, full sync, window filter, resume skip, prune snapshot. No live Miro. Run `bash build.sh --test ./internal/syncer/connector`.
7. Frontend: `DataSourceKey`, card, form, defaults, `en.ts` (and `zh.ts` if that is the locale pair), icon, category sentence in `docs/guides/data_source/data_source_categories_and_selection.md`. `syncDeletedFiles` only if prune returns a real full snapshot.

Defer:

- In-app OAuth redirect, client secret storage, and refresh. Refresh is required only for apps created with expiring tokens, and the syncer has no token-refresh store today. Notion does not do this dance.
- Per-item documents (stickies, cards, text, shapes).
- Image and uploaded-document bytes.
- Enterprise PDF/HTML/SVG export (`boards:export`, Company Admin, eDiscovery).
- Webhooks and the experimental subscription endpoint.
- A Python `common/data_source` connector, unless the product decision is that python/hybrid must serve Miro too.

## 5. Open questions

These are not answered by the fetched Miro docs or by the repo.

1. Must a new source work on `API_PROXY_SCHEME=python` and `hybrid`? Those schemes still run `rag/svr/sync_data_source.py` and `build_connector_for_source`. The syncer guide tells new work to stay on the Go interface and not add a second implementation. Shipping Go-only means Miro appears in the shared UI and fails test-connection and sync on the Python worker.
2. `GET /v2/boards` and the item GET OpenAPI documents 400, 404, and 429, not 401 or 403. 401 `tokenNotProvided` and 403 `forbiddenAccess` are documented on the enterprise export API. What status a revoked or scope-missing token actually returns on `GET /v2/boards` is not in the list-boards spec. Validate copy should follow the status that call returns, once observed.
3. Get boards describes 404 as "Invalid access" with code `notFound`. It is unclear when Miro uses 404 versus a missing board versus a private board whose contents cannot be read.
4. Is there a cap on unfiltered `GET /v2/boards` offset paging? The 10,000 cap is stated only when `query` is set.
5. Is `sort=last_modified` stable enough to resume from, and only when `team_id` or `project_id` is present, as the parameter description says? Without that, the connector must impose its own order.
6. Do board and item timestamps always match RFC3339, or do they sometimes use the space-separated example `2022-03-30 17:26:50+00:00`? Notion's parser (`parseNotionTime`) accepts RFC3339Nano and treats a parse failure as the zero time, which then passes the window check.
7. `FrameData`, `doc_format`, `data_table_format`, and `preview` have no field list in the fetched get-items schema. What text they contain, and whether a type-specific GET returns more than the list call, is unread.
8. The list-items prose mentions child items, and a sample link includes `parent_item_id`, but that parameter is not in the published parameter list. Whether frames must be expanded with a second call is unknown.
9. Item `modifiedAt` has no server-side filter. Item-level incremental sync means listing every item on every board (Level 2) and filtering locally. No doc says whether `modifiedAt` updates for a move, a color change, or only a text edit.
10. Board ids in examples contain `=`. No doc says whether the id is case-sensitive or stable across copy/move between teams.
11. Webhook subscriptions are per board on `/v2-experimental/`. No fetched page says they cover board create/delete, or that a subscription survives token refresh. The syncer has no webhook receiver, so this stays deferred even if the API is real.
12. Non-expiring tokens never refresh. Expiring tokens die in one hour and the refresh token rotates. The repo has OAuth helpers for Google Drive, Gmail, and Box (`web/src/pages/user-setting/data-source/component/`), not a generic refresh store. Which Miro token mode RAGFlow should require is a product choice.
13. `boards:export` is Enterprise, Company Admin, eDiscovery. It is the wrong default for "export the board as a document" for a normal user.
14. The PR template does not require a test plan. `contributing.md` does require test cases for a new feature, and the syncer README lists the cases (window, resume, prune, 401/403/404/429). Miro's list-boards spec does not document 401/403, so those branches should be tested only after the status is confirmed, or tested against the statuses the probe is written to recognize.

## Pages fetched and pages that failed

Fetched and used:

- https://developers.miro.com/reference/api-reference
- https://developers.miro.com/docs/rest-api-build-your-first-hello-world-app
- https://developers.miro.com/reference/authorization-flow-for-expiring-tokens
- https://developers.miro.com/reference/create-authorization-request-link
- https://developers.miro.com/reference/exchange-authorization-code-with-access-token
- https://developers.miro.com/reference/get-new-access-token-using-refresh-token
- https://developers.miro.com/reference/authorization-model
- https://developers.miro.com/reference/scopes
- https://developers.miro.com/reference/rate-limiting
- https://developers.miro.com/reference/get-boards-1
- https://developers.miro.com/reference/get-items-1
- https://developers.miro.com/reference/get-specific-item-1
- https://developers.miro.com/reference/rest-api-item-model
- https://developers.miro.com/reference/enterprise-create-board-export
- https://developers.miro.com/reference/board-content-logs
- https://developers.miro.com/docs/getting-started-with-webhooks-python
- https://developers.miro.com/docs/set-up-a-test-endpoint-for-webhooks
- https://developers.miro.com/docs/getting-started (navigation only)

Could not fetch:

- https://developers.miro.com/docs/web-sdk-vs-rest-api — HTTP 404
- https://developers.miro.com/reference/permission-scopes — HTTP 404. The scopes page that responded is https://developers.miro.com/reference/scopes

No separate "Create webhook subscription" reference page was found. The experimental POST is documented on the Pipedream guide above.
