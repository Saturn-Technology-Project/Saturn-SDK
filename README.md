# Saturn SDK

[中文](#saturn-sdk-中文)

SDKs for driving [Saturn](https://github.com/Saturn-Technology-Project/Saturn-App), the local-first agent runtime, desktop app and IDE, from code: start agent runs, follow their steps as they happen, answer approvals and read what changed. They talk to the Saturn running on the same computer over its local HTTP API with the runtime token, like Saturn's own apps and its `saturn` command line; Saturn must be open.

| Package | Language | Where |
|---|---|---|
| `@saturn-app/sdk` | TypeScript / JavaScript (Node 18+) | [`packages/typescript`](packages/typescript) |

Not published to a registry yet: build from source (`npm install && npm run build` in the package's folder).

```ts
import { Saturn } from "@saturn-app/sdk";

const saturn = new Saturn(); // finds the running Saturn and its token
const run = await saturn.run("fix the failing test", { agent: "Coder", project: "/path/to/repo" });
console.log(run.status, run.output);
```

An agent on Claude (via Claude Code), a personal plan, is refused: Anthropic's terms allow automated access to Claude only with an API key. Use an agent on an API model.

## Development

Each package has its own folder, lock file and scripts. The TypeScript SDK's tests start a real Saturn runtime from a [Saturn-App](https://github.com/Saturn-Technology-Project/Saturn-App) checkout (beside this one, or `SATURN_APP_DIR`) with a scripted model, so they need no credentials:

```bash
git clone https://github.com/Saturn-Technology-Project/Saturn-App ../Saturn-App && npm ci --prefix ../Saturn-App/runtime
cd packages/typescript && npm install && npm run typecheck && npm test
```

Contributions follow Saturn's [contribution guide](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CONTRIBUTING.md), [code of conduct](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CODE_OF_CONDUCT.md) and [governance](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/GOVERNANCE.md). Report vulnerabilities as [`SECURITY.md`](SECURITY.md) says. MIT licensed.

---

# Saturn SDK（中文）

[English](#saturn-sdk)

用程式碼操作 [Saturn](https://github.com/Saturn-Technology-Project/Saturn-App)（本機優先的 Agent Runtime、桌面應用程式與 IDE）的 SDK：啟動 Agent 的 run、即時取得每個步驟、回答核准、讀取它改了什麼。SDK 透過本機 HTTP API 與 Runtime token 連到同一台電腦上執行中的 Saturn，與 Saturn 自己的 App 與 `saturn` 指令相同；Saturn 必須開著。

| 套件 | 語言 | 位置 |
|---|---|---|
| `@saturn-app/sdk` | TypeScript / JavaScript（Node 18 以上） | [`packages/typescript`](packages/typescript) |

尚未發布到套件庫：請從原始碼建置（在套件資料夾中執行 `npm install && npm run build`）。

使用 Claude（透過 Claude Code）訂閱方案的 Agent 會被拒絕：Anthropic 的條款只允許以 API 金鑰自動化存取 Claude。請使用 API 模型的 Agent。

## 開發

每個套件有自己的資料夾、lock 檔與指令。TypeScript SDK 的測試會從 [Saturn-App](https://github.com/Saturn-Technology-Project/Saturn-App) 的 checkout（放在本 repo 旁邊，或以 `SATURN_APP_DIR` 指定）啟動真的 Saturn Runtime，搭配腳本化的模型，不需要任何憑證。

貢獻請遵守 Saturn 的[貢獻指南](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CONTRIBUTING.md)、[行為準則](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/CODE_OF_CONDUCT.md)與[專案治理](https://github.com/Saturn-Technology-Project/Saturn-App/blob/main/GOVERNANCE.md)。安全漏洞請依 [`SECURITY.md`](SECURITY.md) 回報。採 MIT 授權。
