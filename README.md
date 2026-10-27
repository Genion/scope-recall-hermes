# Rune Scope Recall 适配维护 fork

本仓库是 [Genion/rune](https://github.com/Genion/rune) 使用的 Scope Recall 源码维护 fork，上游为 [410979729/scope-recall-hermes](https://github.com/410979729/scope-recall-hermes)。Rune 通过自己的宿主桥接使用 Core；更新方式是审查上游、完成宿主适配、验证后固定源码快照。

**Rune 的更新入口不是 `pip install -U hermes-scope-recall`，也不是上游的 `setup` / `apply-install`。** 这些命令面向下文的 Hermes、Codex 等独立插件安装。不要把它们用于 Rune，不要向 Rune 写入其他宿主的 wrapper、MCP、hooks、安装回执或共享记忆登记。

## Rune 集成边界

- 此 fork 保存完整上游源码及必要维护修复。Rune 固定快照位于 `engine/scope-recall/`，来源提交、真实 Git tree、下载包 SHA-256 和逐文件校验保存在 `engine/scope-recall-source.json`。
- Rune 适配层保留在 Rune 仓库：`engine/scripts/scope-recall-bridge.py` 负责 Core 身份、范围与操作映射；`engine/src/scope-recall.mjs` 负责进程、模型调用、预算与调度。不能把 Rune 的业务规则塞入通用 Core，不能把 Hermes/Codex 的宿主适配器直接当作 Rune 入口。
- 当前 Core 仍使用 schema 1110。Rune 记忆按宿主登记的项目 ID / 工作区 ID 隔离；升级源码不能自动导入历史聊天、合并记忆库、修复真实数据或启用 Embedding。存量数据维护必须先核对当前资料目录和所需动作，再按单独授权执行。
- Core 源码版本与 Python 依赖环境版本是不同记录。源码更新先核对基础依赖锁；锁未变化时复用已验证的依赖环境。不能仅为让版本号一致而重新安装上游包或改写运行环境回执。

## 维护与更新顺序

1. 在独立源码仓库核对 `main`、已有修改、`origin` 与 `upstream`；固定准备审查的上游完整提交，不在 Rune 内克隆嵌套仓库。
2. 阅读上游差异与升级说明，检查 Rune 消费的 Core 接口、身份和证据可见性、事件来源、捕获状态、召回包、失败重试、预算、队列、删除与恢复契约；先复现相关问题，再实现必要的最小适配。
3. 按实际变动验证 Core 与宿主边界。在临时测试资料中验收捕获、提炼、跨会话召回、跨项目隔离、预算持久化、失败处理、归档/删除和恢复；测试包安装只能用于独立验收环境，不能替代 Rune 的源码更新。新增依赖或真实资料维护须另行说明并确认。
4. 在此 fork 的 `main` 提交必要修复与文档。远程推送须单独授权，正常推送后读回完整提交号；不强推、不重写上游历史。
5. 在 Rune 仓库用既有同步脚本同步这个已验证的固定 fork 提交，校验源码及来源清单。未授权推送时，可从 origin/upstream 身份正确、main 干净且 HEAD 等于固定提交的独立仓库通过 `--local-repository` 离线导入；清单必须明确记录本地来源和尚未发布，不能冒充 GitHub 可下载提交。同步前发现快照内有修改时停止，先把修复转入 fork，不强制覆盖。
6. 同步 Rune 的运行版本说明、记忆管理文档和适用回归；最后按已确认时机用 Rune 的正常开发入口启动并验证实际加载。源码同步通过不等于普通用户窗口已加载更新。

当前审查基线为正式版 `3.10.6`，提交 [`8d0f62c906213c2959e18fd5a05dc7fb8ab702d6`](https://github.com/410979729/scope-recall-hermes/commit/8d0f62c906213c2959e18fd5a05dc7fb8ab702d6)（`v3.10.6`）；schema 保持 1110，基础及可选依赖声明、`uv.lock` 均未变化。Rune 直接复用 Core 的人工失败重试：除三类终态拒绝外均可重新打开，能越过被跳过的旧记录，重开后从新的尝试次数开始；源码升级本身不执行真实失败重试。Rune bridge 继续保留部分提炼反馈、指定模型不可用时等待配置修复及自己的持久预算和请求时限。Core 的可选上下文时间参数仍使用默认未知值，独立宿主的提醒/建议过滤、HTTP 模型路由、共享授权和向量诊断不自动成为 Rune 新设置。Rune 实际固定的 fork 提交以其来源清单为准，远程发布状态与普通进程加载结果分别核验。

Rune 的具体命令和验证边界见 [源码同步与适配](https://github.com/Genion/rune/blob/main/docs/scope-recall-sync.md)、[记忆管理](https://github.com/Genion/rune/blob/main/docs/memory-management.md) 和 [运行环境](https://github.com/Genion/rune/blob/main/docs/runtime-environments.md)。

---

以下保留上游的通用说明与独立宿主安装文档；其安装、迁移与部署步骤不用于 Rune 的源码集成。

# Scope Recall

Scope Recall v3 is a bounded local memory core with SQLite as the authority and rebuildable vector companions. It provides host adapters for Hermes, and for Codex, Claude Code, WorkBuddy and DeepSeek Harness (dsh), which share one adapter of hooks and an MCP server (the MCP tools need the optional `codex` extra; dsh runs the hooks through a plugin). The public package is `hermes-scope-recall`; the Python import is `scope_recall`; the host wrapper identity remains `scope-recall`.

This checkout is `3.15.1`. Hermes, Codex, Claude Code, WorkBuddy and dsh can keep one memory: each attaches to a shared
store as an entry, what the owner tells one of them another can recall, and each memory says
which agent it came in through ([docs/shared-store.md](docs/shared-store.md)). An agent that is not
attached keeps its own store. A tool's output is kept and found, but not turned into facts. Each release's notes
are a line or two in [CHANGELOG.md](CHANGELOG.md) and in full on its
[GitHub release](https://github.com/410979729/scope-recall-hermes/releases). Upgrading from `3.2.x` or later to `3.15.1` is
`pip install -U`, `apply-install` and a host restart; the first store open adds the local management audit and private-interval tables. Back up
before upgrading and follow the [management migration and rollback guide](docs/manage.md#migration-and-rollback).
On a shared store in
the shipped embedding space, set `vector_threshold` from 0.653 to 0.70 by hand
([docs/configuration.md](docs/configuration.md#vector_threshold)). Older supported stores migrate in order to schema 1112;
an earlier release cannot open the upgraded store. Section 9 of the
[3.1.0 release notes](https://github.com/410979729/scope-recall-hermes/releases/tag/v3.1.0) is the
migration procedure for a 2.0.1 memory database.
Facts are drawn from accumulated evidence rather than judged one sentence on
sight; only the owner's explicit correction or confirmation changes a fact as it
is captured. SQLite is the only fact authority; host adapters share the same
contracts.

Automatic recall injects zero to six items that clear a lexical coverage or
vector similarity floor. Background preferences and tasks accompany passing
query evidence; a session's first resume request can recover its current task.
Acknowledgments, approvals, praise, bare continuation and session status prompts
within an existing session start no search or query embedding. Explicit recall
keeps its admission behavior. The thresholds and diagnostic counts are described
in [the recall configuration](docs/configuration.md#vector_threshold).

**Recall-quality checks.** The [synthetic recall benchmark](CONTRIBUTING.md#the-recall-benchmark)
provides reproducible development and held-out splits for retrieval, answerability,
scope isolation, packet size and latency. It uses proposed claims and a word-hashing
embedder without a model or network, so it does not establish consolidation quality,
semantic embedding quality or performance on private conversations. The test tiers
also check contracts, storage and host integration.

## Manage your memories

Use `scope-recall ui` with the same store selection to open the [local management page](docs/ui.md).
It provides Chinese and English browsing, reviewed actions, private rules, export and backup status in your browser.

Use `scope-recall memory` to browse, search, inspect history, correct a fact, mute it or delete a conversation.
It runs as the local store owner and makes no model request. Deletion shows the dependent sources and facts before
`--yes` applies it. `scope-recall memory export --out <new directory or .zip>` writes the visible store as a portable
package with original sources, facts, history, episodes, readable Markdown and optional retained attachments.
`scope-recall memory secrets` lists stored credential matches by reference, provenance, time and pattern names.
After upgrading, review its findings, remove them with `scope-recall memory forget <refs>` and change any credential found.
[The management guide](docs/manage.md) covers store selection, filters, version checks and receipts; the
[export guide](docs/export.md) covers package layout, hashes and filtering.

Use `scope-recall private on --session <id> | --project <path> | --all` to stop keeping a session,
directory or the whole store. `--mode full` refuses capture, recall and background model work;
`--mode read-only` refuses capture while recall continues; `--all --for 2h` sets a duration and `off`
closes a rule. Rules are stored as metadata only, overlap takes the stricter mode, and an event is
refused by its own occurrence time even when a later retry or transcript read arrives after the
rule ends. Undated events are refused when earlier private history could cover them. Directory
identity follows junctions and symbolic links on the owning machine; recalls and outbound
requests check again before release. [The private-mode guide](docs/private-mode.md) lists every host path, injection notice,
migration and what is stored or sent in each mode.

## For agents: install or upgrade on the user's behalf

Users only need to ask "install Scope Recall" or "帮我升级一下 scoperecall".
Start with `scope-recall setup --host <hermes-or-codex> --home <actual-instance-home>`.
New users go directly to installation; only detected legacy databases go through
[the agent migration workflow](maintenance/AGENT_WORKFLOW.md). The installed
`scope-recall-setup` skill makes this routing discoverable in Hermes and Codex.
A second installed skill, `scope-recall-memory`, which Claude Code gets too, is for the owner's everyday questions:
what is remembered about me, did I say it or was it worked out, does it still hold, and
what correcting, muting or deleting one memory does before it is done. Claude Code, WorkBuddy and dsh only read
memory through their tools; the change is made in Codex or a Hermes chat on the same store.
Perform path discovery, backup, audience binding, migration, indexing and host
checks yourself; do not ask the user to execute commands or govern old memories.

## Install

Step-by-step Hermes and Codex instructions: [docs/install.md](docs/install.md). Claude Code
installs only as an entry of a shared store, and Codex can join one too:
[docs/shared-store.md](docs/shared-store.md). WorkBuddy runs the same hooks and MCP server, also
only as an entry; its installer adds them to WorkBuddy's own `settings.json` and `mcp.json`
([docs/install.md](docs/install.md), section 12). dsh (DeepSeek Harness) runs them through a plugin
the installer writes and names in dsh's home patch, also only as an entry, with dsh's upload of its
session logs switched off ([docs/install.md](docs/install.md), section 13).

The package is `hermes-scope-recall` on PyPI. Install or upgrade it, with its vector extra, into
the same isolated Python environment the host uses. The default vector store is LanceDB: without
`[lancedb]` it cannot open, and recall finds memories by their words alone (the
`sqlite-bruteforce` backend needs no extra):

```text
python -m pip install -U "hermes-scope-recall[lancedb]"
python -m pip install -U "hermes-scope-recall[lancedb,codex]"
```

The second line is for Codex, Claude Code, WorkBuddy and dsh, whose MCP server needs the `codex`
extra. `-U` moves an older install to the current release; write `==<version>` instead to pin
one. A Hermes that builds its own plugin environment
installs the core from the declaration the installer writes instead (below). `doctor` names an
environment that lacks the vector extra while a LanceDB store is configured
(`vector_dependencies_missing`).

The same wheel and sdist are attached to each
[GitHub release](https://github.com/410979729/scope-recall-hermes/releases/latest)
alongside `SHA256SUMS` and `RELEASE-PROVENANCE.json`, for an offline install
(`python -m pip install "<path-to-wheel>[lancedb]"`). To build it yourself from this checkout instead:

```text
python -m build --wheel
```

Upgrading from 2.0.x is not an in-place upgrade. Read section 9 of the
[3.1.0 release notes](https://github.com/410979729/scope-recall-hermes/releases/tag/v3.1.0) before you start; your old database needs a
one-time offline migration and there are two errors people commonly hit.

The two console names `scope-recall` and `hermes-scope-recall` invoke the same v3 maintenance CLI. They are aliases for the current CLI only; neither is a compatibility promise for an older command set.

Use explicit absolute paths for installation planning. Hermes `--agent-id` must match the host active profile (`get_active_profile_name()`, commonly `default` on an isolated home). Hermes default `--agent-workspace` is `hermes` to match the host memory-provider init contract; pass the same value on plan and apply if you override it. Codex does not accept `--agent-workspace`.

If you talk to Hermes through the Desktop app or `hermes --tui` rather than the CLI, add `--local-platform desktop` (or `tui`) to both commands. Those surfaces name no user unless a dashboard login exists, and a session that names no user is refused everywhere but the CLI until the installer approves the surface; where Hermes's gateway runs those conversations over its local session transport, it names the account the gateway runs as, which binds as the owner on an approved surface and binds nothing on another. [docs/install.md](docs/install.md) says what the approval does and does not cover. If you reach the host through a dashboard login instead (`hermes serve` with `dashboard.basic_auth`, the Desktop app on another machine), the session is that login: approve it as yours with `--owner-login desktop=basic:<name>`. A Hermes with its package
manager (Hermes Desktop, and gateways on a Hermes that has one) builds the Python environment it runs plugins in, and
builds it again on updates: pass it `--target-plugin-dir <home>\plugins\scope-recall`, where Hermes finds the plugin
and the core it declares, vector dependencies included, then let Hermes sync that environment and restart it
(sections 1 and 4 of [docs/install.md](docs/install.md)).

```text
scope-recall plan-install --host hermes --target-plugin-dir <absolute-plugin-dir> --instance-root <absolute-instance-root> --project-root <absolute-project-root> --agent-id <agent-id> --python <absolute-python>
scope-recall apply-install --host hermes --target-plugin-dir <absolute-plugin-dir> --instance-root <absolute-instance-root> --project-root <absolute-project-root> --agent-id <agent-id> --python <absolute-python>
scope-recall doctor --host hermes --instance-root <absolute-instance-root> --python <absolute-python>
```

Codex uses the same commands with `--host codex`, plus `--env-file <absolute-file>` so the Codex-launched MCP server and hooks can read the credential names the runtime config declares (Codex does not pass them in the environment). The installer writes only its own wrapper and installation records and preserves foreign host files. Its receipt records host registration and applicable hook trust as pending at installation time; use doctor and actual host loading to verify the current state. Uninstall keeps Core data by default; explicit purge is bounded to verified installation-owned data and refuses uncertain ownership or active writers.

## Uninstall (default: retain memory)

Uninstall is receipt-driven. By default it removes only plugin wrapper files and retains Core data. Inspect the plan before applying:

```text
scope-recall plan-uninstall --instance-root <absolute-instance-root>
scope-recall apply-uninstall --instance-root <absolute-instance-root>
```

`--target-plugin-dir` may be omitted when the install receipt records it. Purge of installation-owned Core data requires a separate `plan-uninstall --purge` inspection and matching `apply-uninstall --purge`; see [docs/install.md](docs/install.md). Do not treat purge as a normal uninstall step.

## Data and host boundaries

SQLite truth lives in the verified Core data directory. Vector indexes are companions and may be rebuilt only through an explicit configured space. The Hermes and Codex adapters bind host identity, installation identity, scopes, and project roots before opening Core. After that binding is verified, an omitted runtime-config argument checks only `<data_directory>/runtime-config.json`; a missing file stays basic plus an explicit capability gap. The file is never generated or discovered from the current directory, parent directories, or credential locations. Optional host wiring must report a capability gap instead of creating or repairing a database.

The `codex` extra adds the MCP SDK. Without that extra, the Core and Hermes paths remain importable. The `lancedb` extra adds the LanceDB vector companion, the default `vector.backend`; without it, configure `sqlite-bruteforce` or recall stays lexical. PostgreSQL/pgvector is outside this v3 distribution; an explicit configuration error directs operators to retain the old installation and use the migration guide.

On Windows, choose a short data directory such as `C:\ScopeRecall\my-agent`. LanceDB appends index, table, and temporary file names to that path. If the resulting native path is too long, the worker reports `native_vector_path_too_long` before starting LanceDB or calling the embedding API. Use a short target directory for a new installation or an offline migration.

## Profile and entity read views

Two shared read-only Core methods, `profile` and `entity`, are exposed as Hermes tools and on Codex MCP. They return a deterministic categorized current-fact view, or an exact one-hop statement view, over admitted consolidated claims only. Raw chat is never silently turned into a profile. Incoming relations match the full scalar `value_text` only and keep recorded conditions and validity. Explicit project-name aliases may resolve when they are already admitted and still live; person aliases are not generalized. A `budget_tokens` value too small for even the minimal truthful envelope is a validation error, not an oversized view. See [docs/profile-entity.zh-CN.md](docs/profile-entity.zh-CN.md).

## Bounded multi-hop evidence paths

Use `trace` when a question needs two or three recorded relationships joined
together. It is read-only, shared by Hermes and Codex, and reuses existing SQLite
fact/evidence/visibility checks. It does not invoke another model or persist
inferred facts. Cross-scope names and conditional relations are not silently
joined. Node, path, time and explicit byte budgets bound its cost. See
[the trace contract and boundaries](docs/trace.zh-CN.md).

## One store for several agents

Several agents can share one store instead of each keeping its own: each attaches as an
entry, every memory is marked with the agent it came in through, and a deletion through
any of them applies to all. Moving the memory to another machine is copying one directory.
Hermes homes attach as entries, and so do Codex and Claude Code
(`attach --host codex|claude-code`). See [docs/shared-store.md](docs/shared-store.md).
WorkBuddy and dsh attach the same way (`attach --host workbuddy|dsh`). Full backup bundles
and disaster recovery restore to fresh destinations are in [docs/backup-restore.md](docs/backup-restore.md).

## Agent-operated migration

Users ask their agent to upgrade. The bundled `scope-recall-setup` skill routes
fresh installs directly to installation and existing Core databases to ordinary
updates. Only legacy SQLite uses a durable migration job. The agent reads
`scope-recall setup --workflow` and performs discovery, verified audience mapping,
backup, conversion, indexing and actual host checks. See
[the agent upgrade guide](docs/upgrade-guide.zh-CN.md).

Migration composes the existing converter, backup helper and worker. It preserves
original source/history/deletion semantics and never re-extracts the whole old
journal. Unsupported formats or unresolved permissions block cutover and preserve
the old installation. Index scheduling and actual live readiness remain separate.

## Tree layout

The package root holds only the entry (`__init__.py`), the version and the protocol contracts. `core/` is the host-independent memory core over SQLite truth; `vector/` the rebuildable vector companions; `adapters/` the Hermes plugin, the clients that use hooks and an MCP server (Codex, Claude Code, WorkBuddy, dsh; `adapters/clients`, with the entry module names installed configurations run kept in `adapters/codex`) and model transport; `runtime/` the background worker, budgets and scheduling; `maintenance/` install, doctor, upgrade and migration behind the operator CLI. Every shipped module is reachable by import from an entry point named in `packaging_hooks/module_inventory.py`; the wheel allowlist is derived from that, not typed.

## Development checks

The clean wheel must be tested outside the source checkout. At minimum, verify both CLI aliases, a read-only doctor result, Core capture and recall against a temporary installation, and the stdlib HTTP helper's bounded invalid-input response. Host registration, real gateway lifecycle, and production data are separate acceptance boundaries.
