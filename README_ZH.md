<div align="center">

# Saturn SDK

**用程式碼操作你電腦上的 Saturn。**

啟動 Agent 的 run、即時取得每個步驟、回答核准、讀取它改了什麼；使用的是 Saturn 自己的 App 與 `saturn` 命令列所用的同一套本機 API。

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![Node](https://img.shields.io/badge/node-%E2%89%A518-339933.svg?logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6.svg?logo=typescript&logoColor=white)

[安裝](#安裝) · [快速開始](#快速開始) · [API](#api-參考) · [錯誤](#錯誤) · [安全](#安全) · [開發](#開發) · [English](README.md)

</div>

---

## 這是什麼

[Saturn](https://github.com/Saturn-Technology-Project/Saturn-App) 是本機優先的 Agent Runtime、桌面應用程式與 IDE。它的 Runtime 是 `127.0.0.1` 上的 HTTP 服務，所有 Saturn 用戶端都以 bearer token 與它溝通。SDK 是這個 API 的另一個用戶端：腳本可以做到 Agent App 在聊天中做的事，只是不需要視窗。

| 套件 | 語言 | 位置 |
|---|---|---|
| `@saturn-app/sdk` | TypeScript / JavaScript（Node 18 以上） | [`packages/typescript`](packages/typescript) |

TypeScript 套件沒有任何相依套件。它使用 Node 內建的 `fetch` 與 web streams，且是 ES module。

### 如何連線

SDK 自己不會啟動任何東西。**Saturn 必須開著**，在同一台電腦上，且至少執行過一次，token 才會存在。

| 項目 | SDK 依序查找的位置 |
|---|---|
| 位址 | `baseUrl` 選項；否則為 `http://127.0.0.1:<埠>`，埠為 `SATURN_RUNTIME_PORT`，否則 `4700` |
| Token | `token` 選項；否則 `SATURN_RUNTIME_TOKEN`；否則 Saturn 資料夾中的 `runtime-token` 檔案 |
| 資料夾（token 檔案所在） | `dataDir` 選項；否則 `SATURN_DATA_DIR`；否則 `~/.saturn-app` |

Token 只在呼叫 `new Saturn()` 時讀取一次；找不到時，建構函式會拋出錯誤。建構函式不送出任何請求；第一個送出請求的呼叫若在該位址沒有回應，會以 *「Saturn is not running」* 失敗。除非傳入 `baseUrl`，主機一律是 `127.0.0.1`：SDK 不讀取 `SATURN_RUNTIME_HOST`。

每個請求都帶有 `Authorization: Bearer <token>`。Saturn 以權限 `0600` 寫入 token 檔案，並在重新啟動後沿用同一個 token，所以 Saturn 重開後腳本仍可繼續使用。

## 安裝

套件尚未發布到 npm。請從原始碼建置：

```bash
git clone https://github.com/Saturn-Technology-Project/Saturn-SDK.git
cd Saturn-SDK/packages/typescript
npm install
npm run build
```

`npm run build` 會把 `src/` 編譯到 `dist/`，也就是套件匯出的內容。接著從該資料夾加入你的專案：

```bash
npm install /path/to/Saturn-SDK/packages/typescript
```

npm 會連結該資料夾，所以拉取更新後要再執行一次 `npm run build`。若要固定的一份，請在該資料夾執行 `npm pack`，再安裝它產生的 `saturn-app-sdk-0.1.0.tgz`。

## 快速開始

Saturn 開著、有一個名為 `Coder` 的 Agent，且該資料夾已加入 Saturn 成為專案並允許存取：

```ts
import { Saturn } from "@saturn-app/sdk";

const saturn = new Saturn(); // 自動找到執行中的 Saturn 與其 token
const run = await saturn.run("fix the failing test", { agent: "Coder", project: "/path/to/repo" });
console.log(run.status, run.output);
```

`run` 會啟動 run，並在它結束時返回。這個 run 是一般的 Saturn 聊天：會出現在 App 中，它的檔案變更也可以在那裡檢視、保留或回復。

更完整的呼叫，印出每個步驟，並在程式碼中回答核准：

```ts
const run = await saturn.run("add a --verbose flag and test it", {
  agent: "Saturn",
  project: process.cwd(),
  onStep: (step) => console.log("·", step.toolId, step.status),
  onApproval: (step) => step.toolId === "code" && String(step.arguments.command).startsWith("npm test"),
});
console.log(run.status, run.output);
console.log((await saturn.changes(run.id)).files.map((file) => file.path));
```

## API 參考

套件匯出兩個類別 `Saturn` 與 `SaturnError`，以及下列型別。全部都在 [`packages/typescript/src/index.ts`](packages/typescript/src/index.ts)。

| 方法 | 回傳 | 用途 |
|---|---|---|
| [`new Saturn(options?)`](#new-saturnoptions) | `Saturn` | 找到 token 與位址。 |
| [`agents()`](#讀取-saturn-的內容) | `Promise<Agent[]>` | 所有 Agent，與 Saturn 側邊欄相同。 |
| [`projects()`](#讀取-saturn-的內容) | `Promise<Project[]>` | Saturn 知道的專案資料夾。 |
| [`runs(limit = 20)`](#讀取-saturn-的內容) | `Promise<RunSummary[]>` | 最近的 run，最新的在前。 |
| [`getRun(runId)`](#讀取-saturn-的內容) | `Promise<Run>` | 單一 run 目前的狀態。 |
| [`run(prompt, options?)`](#啟動-run) | `Promise<Run>` | 啟動 run 並等它結束：多數腳本只需要這一個呼叫。 |
| [`start(prompt, options?)`](#啟動-run) | `Promise<Run>` | 啟動 run 並立即返回。 |
| [`stream(runId, signal?)`](#追蹤-run) | `AsyncGenerator<Run>` | run 每次變化時的內容，直到它結束或等待核准。 |
| [`wait(runId, options?)`](#追蹤-run) | `Promise<Run>` | 追蹤 run 直到結束，並以 `onApproval` 回答核准。 |
| [`decide(runId, actionId, allow)`](#核准) | `Promise<void>` | 允許或拒絕等待核准的步驟。 |
| [`changes(runId)`](#變更與-diff) | `Promise<{ root, files }>` | run 改了哪些檔案，以及它們的 diff。 |
| [`stop(runId)`](#取消) | `Promise<Run>` | 取消 run。 |

`saturn.baseUrl`（唯讀）是目前使用的位址，結尾沒有斜線。

### `new Saturn(options?)`

```ts
type SaturnOptions = {
  baseUrl?: string; // 預設 http://127.0.0.1:<SATURN_RUNTIME_PORT 或 4700>
  token?: string;   // 預設 SATURN_RUNTIME_TOKEN，否則讀 token 檔案
  dataDir?: string; // token 檔案所在的資料夾；預設 SATURN_DATA_DIR，否則 ~/.saturn-app
};
```

預設值適用於 Saturn App。若要連到以其他埠或其他資料夾啟動的 Runtime（例如為測試啟動的），請傳入選項：

```ts
const saturn = new Saturn({ baseUrl: "http://127.0.0.1:4701", dataDir: "/tmp/saturn-test-data" });
```

### 讀取 Saturn 的內容

```ts
for (const agent of await saturn.agents()) console.log(agent.name, agent.id);
for (const project of await saturn.projects()) console.log(project.name, project.path);
for (const item of await saturn.runs(5)) console.log(item.status, item.agentName, item.threadTitle);
```

`runs(limit)` 會回傳所有仍在進行中的 run（排隊中、執行中、等待核准或休眠中），再加上最多 `limit` 個已結束的 run，最新的在前。子 Agent 的 run 與群組對話的回合不包含在內。`getRun(runId)` 回傳完整的 run，包含它的步驟，以及放在 `children` 中的子 Agent run。

### 啟動 run

`run(prompt, options)` 與 `start(prompt, options)` 接受相同的選項；`start` 會忽略 `onStep` 與 `onApproval`。

```ts
type RunOptions = {
  agent?: string;                 // Agent 的名稱（不分大小寫）或 id
  project?: string;               // Saturn 專案，以資料夾路徑或 id 指定
  chat?: string;                  // 延續既有的聊天，而不是開新的
  mode?: "agent" | "ask" | "plan";
  onStep?: (action: RunAction, run: Run) => void;
  onApproval?: (action: RunAction, run: Run) => boolean | Promise<boolean>;
  signal?: AbortSignal;
};
```

| 選項 | 預設值 | 說明 |
|---|---|---|
| `agent` | 唯一的 Agent | Saturn 有多個 Agent 時必填。 |
| `project` | 無：run 在 Saturn 自己的工作區中進行 | 相對路徑以目前目錄為基準解析。該資料夾必須已是 Saturn 專案，且已在 Saturn 中允許存取。指定 `chat` 時會忽略，因為聊天已有自己的資料夾。 |
| `chat` | 新的聊天 | 聊天的 id，也就是 run 的 `threadId`。 |
| `mode` | `"agent"` | `"agent"` 可以修改內容。`"ask"` 與 `"plan"` 只讀取：前者回答問題，後者寫出計畫。 |
| `onStep` | 無 | 見[追蹤 run](#追蹤-run)。 |
| `onApproval` | 無 | 見[核准](#核准)。 |
| `signal` | 無 | 停止追蹤 run；見[取消](#取消)。 |

在啟動任何東西之前，`start` 會確認提示不是空白、找到 Agent、拒絕使用 Claude（透過 Claude Code）的 Agent（[原因](#claude透過-claude-code的-agent)），並找到專案。任何一步失敗都會拋出 [`SaturnError`](#錯誤)，說明該怎麼做。它回傳的 run 才剛開始，狀態通常是 `running`。

要延續聊天，請傳入先前 run 的 `threadId`：

```ts
const first = await saturn.run("explain src/index.ts", { agent: "Saturn", project: process.cwd(), mode: "ask" });
const next = await saturn.run("now list its exports", { agent: "Saturn", chat: first.threadId, mode: "ask" });
console.log(next.output);
```

### 追蹤 run

`stream(runId, signal?)` 會在 run 每次變化時產出**整個 run**：先是你呼叫當下的狀態，之後每次變化各一次，子 Agent 的變化也算在內。只要某個值的狀態不是 `queued` 或 `running`，它就在產出該值後結束；所以 run 完成時會結束，停下來等待核准時也會結束。對已結束的 run，它只產出一次。

```ts
const started = await saturn.start("summarize the README", { agent: "Saturn", project: process.cwd(), mode: "ask" });
for await (const run of saturn.stream(started.id)) {
  console.log(run.status, `${run.actions.length} steps`);
}
```

沒有另外的事件型別：每個值都是一個 `Run`，最後一個就是 run 停下時的狀態。SDK 底層讀取的是 Saturn 在 `GET /runs/:id/events` 提供的 server-sent events；追蹤途中若 run 被刪除，stream 會拋出 `SaturnError`。

`wait(runId, { onStep, onApproval, signal })` 以 `stream` 追蹤 run 直到結束。run 每走一步，它就依序呼叫一次 `onStep(action, run)`；等待核准的步驟會在得到回答之後，以最終狀態回報。子 Agent 執行的步驟在 `run.children` 中，不會傳給 `onStep`。每次呼叫 `wait` 都從第一步開始計算，所以對同一個 run 再呼叫一次，會再次回報先前的步驟。

`run(prompt, options)` 就是先 `start` 再 `wait`。

run 的狀態是下列其中之一：

| 狀態 | 意義 | `wait` 的行為 |
|---|---|---|
| `queued`、`running` | 進行中 | 繼續追蹤 |
| `waiting_approval` | 有步驟在等你回答 | 詢問 `onApproval`；沒有時直接回傳 run |
| `sleeping` | 因電腦操作的長時間等待而暫停；Saturn 會自行恢復 | 回傳 run；恢復後再呼叫一次 `wait` |
| `completed` | 已完成；答案在 `output` | 回傳 run |
| `failed` | 以錯誤結束，錯誤在 `error` | 回傳 run |
| `cancelled` | 以 `stop()` 或在 Saturn 中停止 | 回傳 run |

`RunStatus` 型別寫的是 `"stopped"`，但 Saturn 回報已停止的 run 為 `"cancelled"`。這個型別接受任何字串，所以請與 `"cancelled"` 比較。

### 核准

當 Agent 的核准策略或權限規則要求詢問時，步驟就會等待核准，就像在 Saturn 中會出現核准卡一樣。從程式碼回答有兩種方式。

**在 callback 中。** 提供 `onApproval`。`wait`（因此 `run` 也是）會以等待中的步驟與 run 呼叫它，送出你的回答（`true` 允許、`false` 拒絕），然後繼續追蹤。它可以回傳 promise。

```ts
const run = await saturn.run("update the changelog", {
  agent: "Careful",
  project: process.cwd(),
  onApproval: async (step) => step.toolId !== "code", // 除了 shell 指令之外都允許
});
```

**手動回答。** 沒有 `onApproval` 時，`run` 與 `wait` 會回傳狀態為 `waiting_approval` 的 run。找到等待中的步驟，以 `decide` 回答，再呼叫一次 `wait`：

```ts
let run = await saturn.run("tidy the imports", { agent: "Careful", project: process.cwd() });
while (run.status === "waiting_approval") {
  const step = run.actions.find((action) => action.status === "waiting_approval");
  if (!step) break;
  console.log("asks to run", step.toolId, step.arguments);
  await saturn.decide(run.id, step.id, true);
  run = await saturn.wait(run.id);
}
```

被拒絕的步驟狀態會是 `denied`；Agent 會被告知並繼續，所以 run 本身通常仍會完成。回答一個已不再等待的步驟，會以 `status` 為 `409` 的 `SaturnError` 失敗。腳本也可以留下等待中的 run 直接結束；核准會留在 Saturn 的收件匣，你可以在 App 中回答。

### 變更與 diff

`changes(runId)` 依 Saturn 的檢查點，回傳 run 在其專案資料夾中改了什麼：

```ts
const { root, files } = await saturn.changes(run.id);
console.log(root); // 專案資料夾；run 沒有在那裡改任何東西時為 null
for (const file of files) console.log(file.status, file.path, `+${file.additions} -${file.deletions}`, file.review);
console.log(files[0]?.diff ?? "no changes");
```

| 欄位 | 內容 |
|---|---|
| `path` | 檔案路徑，相對於 `root` |
| `status` | `added`、`modified` 或 `deleted` |
| `additions`、`deletions` | 新增與刪除的行數 |
| `diff` | unified diff（`diff -u` 格式）；二進位檔則是一行說明它已變更。過長的 diff 會被 Saturn 截斷。 |
| `review` | `pending`、`kept` 或 `reverted`：在 Saturn 中對這項變更的決定 |

shell 指令改動的檔案也包含在內。SDK 只讀取變更：保留或回復請在 Saturn 中進行，或使用 Saturn [本機 API](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/README_ZH.md#本機-api) 的 `POST /runs/:id/changes/keep` 與 `/changes/revert`。

### 取消

`stop(runId)` 會在 Saturn 中取消 run 及其子 Agent，並回傳狀態為 `cancelled` 的 run。已結束的 run 會原樣回傳。若你正在追蹤該 run，`wait` 與 `stream` 隨後會以該狀態結束：

```ts
const started = await saturn.start("refactor the parser", { agent: "Saturn", project: process.cwd() });
const timer = setTimeout(() => void saturn.stop(started.id), 5 * 60_000);
const run = await saturn.wait(started.id);
clearTimeout(timer);
console.log(run.status); // 超過五分鐘就是 "cancelled"
```

`signal` 選項不一樣：中止它只會讓**你的腳本**停止追蹤 run，而不是停止 run。此時 `stream` 與 `wait` 會以 `fetch` 的 `AbortError` 拒絕（不是 `SaturnError`），run 則在 Saturn 中繼續進行。兩者都要的話，先中止再停止：

```ts
const controller = new AbortController();
process.once("SIGINT", () => controller.abort());
try {
  await saturn.wait(started.id, { signal: controller.signal });
} catch (error) {
  if ((error as Error).name !== "AbortError") throw error;
  await saturn.stop(started.id);
}
```

`start` 不使用 signal，所以在 `run` 中，它要等 run 啟動之後才會生效。

### 錯誤

失敗會以 `SaturnError` 拋出。它是 `Error` 的子類別，有一個選用的 `status`：Saturn 回應錯誤時是 HTTP 狀態碼，SDK 在送出請求前就拒絕時則是 `undefined`。訊息是 Saturn 自己的（其 JSON 回應中的 `error`），或是 SDK 寫的，說明該怎麼做。

```ts
import { Saturn, SaturnError } from "@saturn-app/sdk";

try {
  const saturn = new Saturn();
  await saturn.run("fix the failing test", { agent: "Coder" });
} catch (error) {
  if (!(error instanceof SaturnError)) throw error;
  console.error(error.message, error.status ?? "no HTTP status");
}
```

| 情況 | 訊息開頭 | `status` |
|---|---|---|
| 找不到 token | `Saturn has not run on this computer yet` | 無 |
| 該位址沒有回應 | `Saturn is not running` | 無 |
| token 錯誤 | Saturn 的回應 | `401` |
| 提示是空白 | `Give the prompt` | 無 |
| Saturn 中沒有 Agent | `There are no agents yet` | 無 |
| 有多個 Agent 但沒給 `agent` 選項 | `Name the agent` | 無 |
| 沒有該名稱或 id 的 Agent | `No agent named` | 無 |
| 使用 Claude（透過 Claude Code）的 Agent | `<name> runs on Claude (via Claude Code), a personal plan` | 無 |
| 該資料夾不是 Saturn 專案 | `<path> is not a Saturn project` | 無 |
| Saturn 尚未取得該專案的存取權 | `Saturn does not have access to` | 無 |
| 沒有該 id 的 run | `Run not found` | `404` |
| 回答已不再等待的步驟 | `This approval is no longer waiting` | `409` |
| 追蹤途中 run 被刪除 | `Run <id> was deleted` | 無 |

有兩種錯誤不是 `SaturnError`：中止 `signal` 時的 `AbortError`，以及你的 `onStep` 或 `onApproval` callback 拋出的任何錯誤，`wait` 會原樣傳出。

### 型別

| 型別 | 欄位 |
|---|---|
| `Run` | `id`、`agentId`、`threadId?`（聊天）、`input`（提示）、`status`、`output`（目前為止的答案）、`error?`、`actions`（依序的步驟）、`children?`（子 Agent 的 run） |
| `RunAction` | `id`、`toolId`、`arguments`、`status`（`completed`、`denied`、`waiting_approval`、`sleeping` 或 `failed`）、`reason?`、`result?` |
| `RunStatus` | 見[狀態表](#追蹤-run) |
| `RunSummary` | `id`、`agentName`、`status`、`input`（已截短）、`threadTitle` |
| `Agent` | `id`、`name`、`modelConnectionId`（或 `null`）、`mcpConnectionIds` |
| `Project` | `id`、`name`、`path`、`access?`（Saturn 可使用該資料夾後才有值） |
| `ModelConnection` | `id`、`name`、`provider`、`model?`；沒有方法回傳它，`start` 讀取它來檢查 Agent 的供應商 |
| `FileChange` | `path`、`status`、`additions`、`deletions`、`review`、`diff` |
| `RunOptions`、`SaturnOptions` | 見上文 |

Saturn 送出的欄位比這些型別列出的多（例如 run 的推理內容）；執行時它們存在，但沒有型別，且可能變動。

## Claude（透過 Claude Code）的 Agent

使用 Claude（透過 Claude Code）訂閱方案的 Agent 會被拒絕：Anthropic 的條款只允許以 API 金鑰自動化存取 Claude。`start` 會在 run 開始前檢查 Agent 的模型連線，並改為拋出 `SaturnError`。請使用 API 模型的 Agent，例如以你自己的 Anthropic API 金鑰使用 Claude 供應商。

這與 Saturn 對自動化任務、以及在腳本中執行 `saturn run` 套用的規則相同。見 Saturn README 的 [Claude 與 Anthropic 的條款](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/README_ZH.md#claude-與-anthropic-的條款)。

## 相容性

SDK 不檢查 Saturn 的版本，程式碼中也沒有指定任何版本。它需要 Runtime 能回應下列路由；這些路由隨 `saturn` 命令列與 SDK 在 Saturn 0.3.0 加入（見[變更紀錄](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CHANGELOG.md)）：

| 方法 | 路徑 | 使用者 |
|---|---|---|
| `GET` | `/agents`、`/projects`、`/connections/models` | `agents`、`projects`、`start` |
| `POST` | `/agents/:id/runs` | `start` |
| `GET` | `/runs?limit=`、`/runs/:id`、`/runs/:id/events`、`/runs/:id/changes` | `runs`、`getRun`、`stream`、`changes` |
| `POST` | `/runs/:id/actions/:actionId/decision`、`/runs/:id/stop` | `decide`、`stop` |

CI 在每次變更時與每週，對 Saturn-App 的 `main` 分支測試 SDK，所以 Saturn 若有不相容的變更，會在那裡出現。

## 安全

- **Token 是秘密。** 任何持有它的程式都能做 Saturn App 能做的一切，包括在你的專案資料夾中執行 Agent。不要印出、記錄、commit 它，也不要放在命令列參數中。與其把 token 複製到 `SATURN_RUNTIME_TOKEN` 或程式碼中，不如讓 SDK 讀取 token 檔案。
- **只用 loopback。** Saturn 只監聽 `127.0.0.1`，並拒絕 `Host` 不是 loopback 的請求。請讓 `baseUrl` 保持在 `127.0.0.1`：你給 SDK 什麼位址，它就把 token 送到哪裡。
- **與 App 相同的控管。** 從 SDK 啟動的 run 與在 Saturn 中的聊天一樣，受 Agent 的核准策略、權限規則、沙箱與資料夾授權限制。`onApproval` 是用你的程式碼做原本由人在核准卡上做的決定，請和你自己一樣嚴格；對所有步驟都回傳 `true` 就是完全自主。

安全漏洞請依 [`SECURITY.md`](SECURITY.md) 私下回報：寄至 [security@saturn-core.com](mailto:security@saturn-core.com)，或使用本 repository 的 [Report a vulnerability](https://github.com/Saturn-Technology-Project/Saturn-SDK/security/advisories/new)，不要以公開 issue 回報。

## 開發

每個套件有自己的資料夾、lock 檔與指令。TypeScript SDK 的測試會從 [Saturn-App](https://github.com/Saturn-Technology-Project/Saturn-App) 的 checkout（放在本 repo 旁邊，或以 `SATURN_APP_DIR` 指定）啟動真的 Saturn Runtime，搭配腳本化的模型，不需要任何憑證：

```bash
git clone https://github.com/Saturn-Technology-Project/Saturn-App ../Saturn-App && npm ci --prefix ../Saturn-App/runtime
cd packages/typescript && npm install && npm run typecheck && npm test
```

| 指令 | 用途 |
|---|---|
| `npm run typecheck` | 對 `src/` 做嚴格型別檢查 |
| `npm run build` | 把 `src/` 編譯到 `dist/` |
| `npm test` | 以 `node --test` 執行 `test/*.test.ts` |

函式庫可在 Node 18 上執行，但測試直接執行 TypeScript，需要 Node 22.18 以上；CI 使用 Saturn-App 的 `Saturn-IDE/.nvmrc` 指定的版本（Node 24）。測試會在空閒的埠上，以暫存資料夾與 token 啟動 Runtime，並在暫存的 git repository 中工作，所以絕不會動到你的 Saturn。沒有已安裝 `runtime/node_modules` 的 Saturn-App checkout 時，測試會略過並說明原因。

[CI](.github/workflows/ci.yml) 以 `--ignore-scripts` 安裝兩邊的套件，做型別檢查、建置、對 Saturn-App 的 `main` 測試，並以 `npm audit signatures` 驗證套件庫簽章。

## 貢獻

貢獻請遵守 Saturn 的[貢獻指南](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CONTRIBUTING.md)、[行為準則](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CODE_OF_CONDUCT.md)與[專案治理](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/GOVERNANCE.md)。安全漏洞請依 [`SECURITY.md`](SECURITY.md) 回報。

## 授權

[MIT](LICENSE) © 2026 Rick Weng
