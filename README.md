<div align="center">

# Saturn SDK

**Drive the Saturn on your computer from code.**

Start agent runs, follow their steps as they happen, answer approvals and read what changed, over the same local API that Saturn's own apps and its `saturn` command line use.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
![Node](https://img.shields.io/badge/node-%E2%89%A518-339933.svg?logo=node.js&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6.svg?logo=typescript&logoColor=white)

[Install](#install) · [Quick start](#quick-start) · [API](#api-reference) · [Errors](#errors) · [Security](#security) · [Development](#development) · [中文](README_ZH.md)

</div>

---

## What it is

[Saturn](https://github.com/Saturn-Technology-Project/Saturn-App) is a local-first agent runtime, desktop app and IDE. Its runtime is an HTTP service on `127.0.0.1` that every Saturn client talks to with a bearer token. The SDK is one more client of that API: a script can do what the agent app does with a chat, without the window.

| Package | Language | Where |
|---|---|---|
| `@saturn-app/sdk` | TypeScript / JavaScript (Node 18 or later) | [`packages/typescript`](packages/typescript) |

The TypeScript package has no dependencies. It uses Node's built-in `fetch` and web streams, and it is an ES module.

### How it connects

The SDK starts nothing itself. **Saturn must be open** on the same computer, and it must have run at least once, so that its token exists.

| What | Where the SDK looks, in order |
|---|---|
| Address | The `baseUrl` option; else `http://127.0.0.1:<port>`, where the port is `SATURN_RUNTIME_PORT`, else `4700` |
| Token | The `token` option; else `SATURN_RUNTIME_TOKEN`; else the file `runtime-token` in Saturn's data folder |
| Data folder (for the token file) | The `dataDir` option; else `SATURN_DATA_DIR`; else `~/.saturn-app` |

The token is read once, when you call `new Saturn()`; if there is none, the constructor throws. The constructor sends no request. The first call that does fails with *"Saturn is not running"* if nothing answers on the address. The host is always `127.0.0.1` unless you pass `baseUrl`: the SDK does not read `SATURN_RUNTIME_HOST`.

Every request carries `Authorization: Bearer <token>`. Saturn writes the token file with mode `0600` and keeps the same token across restarts, so a script keeps working after Saturn is reopened.

## Install

The package is not published to npm yet. Build it from source:

```bash
git clone https://github.com/Saturn-Technology-Project/Saturn-SDK.git
cd Saturn-SDK/packages/typescript
npm install
npm run build
```

`npm run build` compiles `src/` to `dist/`, which is what the package exports. Then add it to your project from that folder:

```bash
npm install /path/to/Saturn-SDK/packages/typescript
```

npm links the folder, so run `npm run build` again after pulling changes. For a fixed copy instead, run `npm pack` in the folder and install the `saturn-app-sdk-0.1.0.tgz` it writes.

## Quick start

With Saturn open, an agent named `Coder`, and the folder added to Saturn as a project with access allowed:

```ts
import { Saturn } from "@saturn-app/sdk";

const saturn = new Saturn(); // finds the running Saturn and its token
const run = await saturn.run("fix the failing test", { agent: "Coder", project: "/path/to/repo" });
console.log(run.status, run.output);
```

`run` starts the run and returns when it ends. The run is an ordinary Saturn chat: it shows in the app, and its file changes can be reviewed, kept or reverted there.

A fuller call, with every step printed and approvals answered in code:

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

## API reference

The package exports two classes, `Saturn` and `SaturnError`, and the types below. All of it is in [`packages/typescript/src/index.ts`](packages/typescript/src/index.ts).

| Method | Returns | What it does |
|---|---|---|
| [`new Saturn(options?)`](#new-saturnoptions) | `Saturn` | Finds the token and the address. |
| [`agents()`](#reading-what-saturn-has) | `Promise<Agent[]>` | The agents, as in Saturn's sidebar. |
| [`projects()`](#reading-what-saturn-has) | `Promise<Project[]>` | The project folders Saturn knows. |
| [`runs(limit = 20)`](#reading-what-saturn-has) | `Promise<RunSummary[]>` | Recent runs, newest first. |
| [`getRun(runId)`](#reading-what-saturn-has) | `Promise<Run>` | One run as it is now. |
| [`run(prompt, options?)`](#starting-a-run) | `Promise<Run>` | Starts a run and waits for it: the one call most scripts need. |
| [`start(prompt, options?)`](#starting-a-run) | `Promise<Run>` | Starts a run and returns at once. |
| [`stream(runId, signal?)`](#following-a-run) | `AsyncGenerator<Run>` | The run each time it changes, until it ends or waits for approval. |
| [`wait(runId, options?)`](#following-a-run) | `Promise<Run>` | Follows a run until it ends, answering approvals with `onApproval`. |
| [`decide(runId, actionId, allow)`](#approvals) | `Promise<void>` | Allows or denies a step that waits for approval. |
| [`changes(runId)`](#changes-and-diffs) | `Promise<{ root, files }>` | The files a run changed, with their diffs. |
| [`stop(runId)`](#cancelling) | `Promise<Run>` | Cancels a run. |

`saturn.baseUrl` (read-only) is the address in use, without a trailing slash.

### `new Saturn(options?)`

```ts
type SaturnOptions = {
  baseUrl?: string; // default http://127.0.0.1:<SATURN_RUNTIME_PORT or 4700>
  token?: string;   // default SATURN_RUNTIME_TOKEN, else the token file
  dataDir?: string; // where the token file is; default SATURN_DATA_DIR, else ~/.saturn-app
};
```

The defaults are right for the Saturn app. Pass options to reach a runtime started on another port or with another data folder, such as one started for tests:

```ts
const saturn = new Saturn({ baseUrl: "http://127.0.0.1:4701", dataDir: "/tmp/saturn-test-data" });
```

### Reading what Saturn has

```ts
for (const agent of await saturn.agents()) console.log(agent.name, agent.id);
for (const project of await saturn.projects()) console.log(project.name, project.path);
for (const item of await saturn.runs(5)) console.log(item.status, item.agentName, item.threadTitle);
```

`runs(limit)` returns every run that is still active (queued, running, waiting for approval or sleeping), then up to `limit` finished ones, newest first. Sub-agent runs and group chat turns are left out. `getRun(runId)` returns the whole run, with its steps and its sub-agents' runs in `children`.

### Starting a run

`run(prompt, options)` and `start(prompt, options)` take the same options; `start` ignores `onStep` and `onApproval`.

```ts
type RunOptions = {
  agent?: string;                 // the agent's name (any case) or id
  project?: string;               // a Saturn project, by folder path or id
  chat?: string;                  // continue an existing chat instead of starting one
  mode?: "agent" | "ask" | "plan";
  onStep?: (action: RunAction, run: Run) => void;
  onApproval?: (action: RunAction, run: Run) => boolean | Promise<boolean>;
  signal?: AbortSignal;
};
```

| Option | Default | Notes |
|---|---|---|
| `agent` | The only agent | Required when Saturn has more than one agent. |
| `project` | None: the run works in Saturn's own workspace | A relative path is resolved from the current directory. The folder must already be a Saturn project, with access allowed in Saturn. Ignored when `chat` is given, since the chat has its folder already. |
| `chat` | A new chat | The chat's id, which is a run's `threadId`. |
| `mode` | `"agent"` | `"agent"` may change things. `"ask"` and `"plan"` only read: one answers, the other writes a plan. |
| `onStep` | None | See [Following a run](#following-a-run). |
| `onApproval` | None | See [Approvals](#approvals). |
| `signal` | None | Stops following the run; see [Cancelling](#cancelling). |

Before it starts anything, `start` checks the prompt is not blank, finds the agent, refuses an agent on Claude (via Claude Code) ([why](#claude-via-claude-code-agents)), and finds the project. Each of these fails with a [`SaturnError`](#errors) that says what to do. The run it returns has just begun, usually with status `running`.

To continue a chat, pass the earlier run's `threadId`:

```ts
const first = await saturn.run("explain src/index.ts", { agent: "Saturn", project: process.cwd(), mode: "ask" });
const next = await saturn.run("now list its exports", { agent: "Saturn", chat: first.threadId, mode: "ask" });
console.log(next.output);
```

### Following a run

`stream(runId, signal?)` yields the **whole run** each time it changes: first as it is when you ask, then after each change, including changes to its sub-agents. It ends after the first value whose status is anything but `queued` or `running`, so it ends both when the run finishes and when it stops to wait for an approval. On a run that has already ended it yields once.

```ts
const started = await saturn.start("summarize the README", { agent: "Saturn", project: process.cwd(), mode: "ask" });
for await (const run of saturn.stream(started.id)) {
  console.log(run.status, `${run.actions.length} steps`);
}
```

There are no separate event types: each value is a `Run`, and the last one is where the run stopped. Underneath, the SDK reads Saturn's server-sent events at `GET /runs/:id/events`; if the run is deleted while you follow it, the stream throws a `SaturnError`.

`wait(runId, { onStep, onApproval, signal })` follows a run with `stream` until it ends. It calls `onStep(action, run)` once for each step of the run, in order, when the step is taken; a step that waits for approval is reported after it has been answered, with its final status. Steps that sub-agents take are in `run.children`, not passed to `onStep`. Each call to `wait` counts from the first step, so calling it again on the same run reports the earlier steps again.

`run(prompt, options)` is `start` followed by `wait`.

A run's status is one of these:

| Status | Meaning | `wait` |
|---|---|---|
| `queued`, `running` | Working | Keeps following |
| `waiting_approval` | A step waits for your answer | Asks `onApproval`, or returns the run if there is none |
| `sleeping` | Paused by a long computer-use wait; Saturn resumes it on its own | Returns the run; call `wait` again after it resumes |
| `completed` | Finished; the answer is in `output` | Returns the run |
| `failed` | Ended with an error, in `error` | Returns the run |
| `cancelled` | Stopped with `stop()` or in Saturn | Returns the run |

The `RunStatus` type names `"stopped"`, but Saturn reports a stopped run as `"cancelled"`. The type accepts any string, so compare with `"cancelled"`.

### Approvals

A step waits for approval when the agent's approval policy or a permission rule says to ask, as it would show an approval card in Saturn. There are two ways to answer it from code.

**In a callback.** Give `onApproval`. `wait` (and so `run`) calls it with the waiting step and the run, sends your answer (`true` allows, `false` denies) and keeps following. It may return a promise.

```ts
const run = await saturn.run("update the changelog", {
  agent: "Careful",
  project: process.cwd(),
  onApproval: async (step) => step.toolId !== "code", // allow everything but shell commands
});
```

**By hand.** Without `onApproval`, `run` and `wait` return the run with status `waiting_approval`. Find the waiting step, answer it with `decide`, then `wait` again:

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

A denied step ends with status `denied`; the agent is told and carries on, so the run itself usually still completes. Answering a step that no longer waits fails with a `SaturnError` with status `409`. A script that leaves the run waiting can stop; the approval stays in Saturn's inbox, where you can answer it in the app.

### Changes and diffs

`changes(runId)` returns what the run changed in its project folder, from Saturn's checkpoints:

```ts
const { root, files } = await saturn.changes(run.id);
console.log(root); // the project folder; null when the run changed nothing there
for (const file of files) console.log(file.status, file.path, `+${file.additions} -${file.deletions}`, file.review);
console.log(files[0]?.diff ?? "no changes");
```

| Field | Value |
|---|---|
| `path` | The file's path, relative to `root` |
| `status` | `added`, `modified` or `deleted` |
| `additions`, `deletions` | Lines added and removed |
| `diff` | A unified diff (`diff -u` style); for a binary file, one line saying it changed. Saturn cuts very long diffs. |
| `review` | `pending`, `kept` or `reverted`: what has been decided about the change in Saturn |

Files that a shell command changed are included. The SDK only reads changes: keep or revert them in Saturn, or with `POST /runs/:id/changes/keep` and `/changes/revert` in Saturn's [local API](https://github.com/Saturn-Technology-Project/Saturn-App#local-api).

### Cancelling

`stop(runId)` cancels the run in Saturn, along with its sub-agents, and returns it with status `cancelled`. A run that has already ended is returned as it is. If you are following the run, `wait` and `stream` then end with that status:

```ts
const started = await saturn.start("refactor the parser", { agent: "Saturn", project: process.cwd() });
const timer = setTimeout(() => void saturn.stop(started.id), 5 * 60_000);
const run = await saturn.wait(started.id);
clearTimeout(timer);
console.log(run.status); // "cancelled" if it ran past five minutes
```

The `signal` option is different: aborting it stops **your script** following the run, not the run. `stream` and `wait` then reject with the `AbortError` from `fetch`, not a `SaturnError`, and the run keeps going in Saturn. To do both, abort and stop:

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

`start` does not use the signal, so in `run` it applies only once the run has started.

### Errors

Failures are thrown as `SaturnError`, a subclass of `Error` with an optional `status`: the HTTP status when Saturn answered with an error, and `undefined` when the SDK refused before asking. The message is Saturn's own (`error` from its JSON answer), or the SDK's, written to say what to do.

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

| When | Message begins | `status` |
|---|---|---|
| No token found | `Saturn has not run on this computer yet` | none |
| Nothing listens on the address | `Saturn is not running` | none |
| The token is wrong | Saturn's answer | `401` |
| A blank prompt | `Give the prompt` | none |
| No agents in Saturn | `There are no agents yet` | none |
| Several agents and no `agent` option | `Name the agent` | none |
| No agent by that name or id | `No agent named` | none |
| An agent on Claude (via Claude Code) | `<name> runs on Claude (via Claude Code), a personal plan` | none |
| The folder is not a Saturn project | `<path> is not a Saturn project` | none |
| Saturn has no access to the project yet | `Saturn does not have access to` | none |
| No run with that id | `Run not found` | `404` |
| Answering a step that no longer waits | `This approval is no longer waiting` | `409` |
| The run was deleted while you follow it | `Run <id> was deleted` | none |

Two errors are not `SaturnError`: the `AbortError` when you abort a `signal`, and whatever your `onStep` or `onApproval` callback throws, which `wait` passes on.

### Types

| Type | Fields |
|---|---|
| `Run` | `id`, `agentId`, `threadId?` (the chat), `input` (the prompt), `status`, `output` (the answer so far), `error?`, `actions` (the steps, in order), `children?` (sub-agents' runs) |
| `RunAction` | `id`, `toolId`, `arguments`, `status` (`completed`, `denied`, `waiting_approval`, `sleeping` or `failed`), `reason?`, `result?` |
| `RunStatus` | See the [status table](#following-a-run) |
| `RunSummary` | `id`, `agentName`, `status`, `input` (shortened), `threadTitle` |
| `Agent` | `id`, `name`, `modelConnectionId` (or `null`), `mcpConnectionIds` |
| `Project` | `id`, `name`, `path`, `access?` (set once Saturn may use the folder) |
| `ModelConnection` | `id`, `name`, `provider`, `model?`; no method returns it, `start` reads it to check an agent's provider |
| `FileChange` | `path`, `status`, `additions`, `deletions`, `review`, `diff` |
| `RunOptions`, `SaturnOptions` | Above |

Saturn sends more fields than these types name (a run's reasoning, for instance); they are present at runtime but not typed, and may change.

## Claude (via Claude Code) agents

An agent on Claude (via Claude Code), a personal plan, is refused: Anthropic's terms allow automated access to Claude only with an API key. `start` checks the agent's model connection before the run begins and throws a `SaturnError` instead. Use an agent on an API model, such as the Claude provider with your own Anthropic API key.

It is the same rule Saturn applies to automations and to `saturn run` in a script. See [Claude and Anthropic's terms](https://github.com/Saturn-Technology-Project/Saturn-App#claude-and-anthropics-terms) in Saturn's README.

## Compatibility

The SDK checks no Saturn version, and the code names none. It needs a Saturn whose runtime answers these routes, which arrived with the `saturn` command line and the SDK in Saturn 0.3.0 ([changelog](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CHANGELOG.md)):

| Method | Path | Used by |
|---|---|---|
| `GET` | `/agents`, `/projects`, `/connections/models` | `agents`, `projects`, `start` |
| `POST` | `/agents/:id/runs` | `start` |
| `GET` | `/runs?limit=`, `/runs/:id`, `/runs/:id/events`, `/runs/:id/changes` | `runs`, `getRun`, `stream`, `changes` |
| `POST` | `/runs/:id/actions/:actionId/decision`, `/runs/:id/stop` | `decide`, `stop` |

CI tests the SDK against Saturn-App's `main` branch on every change and weekly, so an incompatible change in Saturn shows up there.

## Security

- **The token is a secret.** It lets any program that holds it do everything Saturn's apps can, including running agents in your project folders. Do not print it, log it, commit it, or pass it on a command line. Prefer letting the SDK read the token file over copying the token into `SATURN_RUNTIME_TOKEN` or your code.
- **Loopback only.** Saturn listens on `127.0.0.1` alone and refuses requests whose `Host` is not loopback. Keep `baseUrl` on `127.0.0.1`: the SDK sends the token to whatever address you give it.
- **The same controls as the app.** A run from the SDK has the agent's approval policy, permission rules, sandbox and folder grants, as a chat in Saturn does. `onApproval` decides with your code what a person would decide on a card, so make it as strict as you would be; returning `true` for everything is full autonomy.

Report vulnerabilities privately, as [`SECURITY.md`](SECURITY.md) says: to [security@saturn-core.com](mailto:security@saturn-core.com) or through this repository's [Report a vulnerability](https://github.com/Saturn-Technology-Project/Saturn-SDK/security/advisories/new), never in a public issue.

## Development

Each package has its own folder, lock file and scripts. The TypeScript SDK's tests start a real Saturn runtime from a [Saturn-App](https://github.com/Saturn-Technology-Project/Saturn-App) checkout (beside this one, or `SATURN_APP_DIR`) with a scripted model, so they need no credentials:

```bash
git clone https://github.com/Saturn-Technology-Project/Saturn-App ../Saturn-App && npm ci --prefix ../Saturn-App/runtime
cd packages/typescript && npm install && npm run typecheck && npm test
```

| Script | What it does |
|---|---|
| `npm run typecheck` | Strict type-check of `src/` |
| `npm run build` | Compiles `src/` to `dist/` |
| `npm test` | Runs `test/*.test.ts` with `node --test` |

The library runs on Node 18, but the tests run TypeScript directly, which needs Node 22.18 or later; CI uses the version in Saturn-App's `Saturn-IDE/.nvmrc` (Node 24). The test starts the runtime on a free port with a temporary data folder and token, and works in a temporary git repository, so it never touches your Saturn. Without a Saturn-App checkout that has `runtime/node_modules`, it is skipped and says why.

[CI](.github/workflows/ci.yml) installs both packages with `--ignore-scripts`, type-checks, builds, tests against Saturn-App's `main`, and verifies registry signatures with `npm audit signatures`.

## Contributing

Contributions follow Saturn's [contribution guide](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CONTRIBUTING.md), [code of conduct](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CODE_OF_CONDUCT.md) and [governance](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/GOVERNANCE.md). Report vulnerabilities as [`SECURITY.md`](SECURITY.md) says.

## License

[MIT](LICENSE) © 2026 Rick Weng
