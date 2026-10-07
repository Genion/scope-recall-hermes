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
4. 在此 fork 的 `main` 提交必要修复与文档，正常推送并读回完整提交号；不强推、不重写上游历史。
5. 在 Rune 仓库用既有同步脚本同步这个已验证的 fork 提交，校验源码及来源清单；同步前发现快照内有修改时停止，先把修复转入 fork，不强制覆盖。
6. 同步 Rune 的运行版本说明、记忆管理文档和适用回归；最后按已确认时机用 Rune 的正常开发入口启动并验证实际加载。源码同步通过不等于普通用户窗口已加载更新。

本轮审查基线为上游 `3.8.0`，提交 `323a4ec7dcc650158a17393206ef9675e16d196b`。Rune 实际固定的 fork 提交以其来源清单为准，不把上游版本号、fork HEAD、依赖环境回执或进程实际加载版本混为一谈。

Rune 的具体命令和验证边界见 [源码同步与适配](https://github.com/Genion/rune/blob/main/docs/scope-recall-sync.md)、[记忆管理](https://github.com/Genion/rune/blob/main/docs/memory-management.md) 和 [运行环境](https://github.com/Genion/rune/blob/main/docs/runtime-environments.md)。

---

以下保留上游的通用说明与独立宿主安装文档；其安装、迁移与部署步骤不用于 Rune 的源码集成。

# Scope Recall 3.8 autonomous memory

Scope Recall v3 is a bounded local memory core with SQLite as the authority and rebuildable vector companions. It provides host adapters for Hermes, Codex and Claude Code (the last two share one adapter of hooks and an MCP server), with the MCP tools when the optional `codex` extra is installed. The public package is `hermes-scope-recall`; the Python import is `scope_recall`; the host wrapper identity remains `scope-recall`.
WorkBuddy runs that same hook adapter and MCP server, as an entry of a shared store, and so does DeepSeek Harness (dsh), through a plugin.

This checkout is `3.8.0`, in which an operator can re-embed a store into a new embedding space (`respace-embeddings`) and the doctor says when embeddings have waited a day, Hermes keeps what a failed tool call printed, an automatic recall reads its evidence in about 470 statements where it ran 16,000, a capture's write no longer grows with its words, its episode or the store's scopes in a busy Hermes gateway (a long tool output held the shared store's writer lease for up to 43 s), a claim's vector work comes back after a provider failed it (`retry-failures` reopens what earlier releases dropped), a withheld tool output's placeholder is indexed by the tool's own error text alone, and what Hermes writes into a conversation itself is stored as the host's, not the owner's, also when a compression folded its summary or a to-do list into it; since 3.7.0 DeepSeek Harness (dsh) joins the shared store too: a dsh plugin recalls before each turn and stores each turn's messages, and dsh's upload of its session logs is switched off ([docs/install.md](docs/install.md), section 13). Hermes, Codex, Claude Code, WorkBuddy and dsh can keep one memory: each attaches to a shared
store as an entry, what the owner tells one of them another can recall, and each memory says
which agent it came in through ([docs/shared-store.md](docs/shared-store.md)). Hermes agents
could share a store from 3.2.0; Codex and Claude Code join in 3.3.0. An agent that is not
attached keeps its own store. A tool's output is still kept and found, but no longer turned
into facts. The notes are the `[3.7.x]`, `[3.6.x]`, `[3.5.x]` and `[3.4.x]` sections of [CHANGELOG.md](CHANGELOG.md), newest first; upgrading
from `3.4.x`, `3.3.x` or `3.2.x` is `pip install -U`, `apply-install` and a host restart, and the store's
schema does not change; on a shared store in the shipped embedding space, set `vector_threshold`
from 0.653 to 0.70 by hand ([docs/configuration.md](docs/configuration.md#vector_threshold)). From `3.1.x` the store moves to schema 1110 the first time it is opened,
after which a 3.1 process cannot open it.
3.1 is a rebuild rather than a patch on 2.0: production
code went from 141,044 lines to 48,289, memory now accumulates evidence before a
fact is written rather than judging one sentence on sight, and hosts sit behind
adapters instead of the core being shaped around Hermes. The notes for the rebuild are
the `[3.1.0]` section of [CHANGELOG.md](CHANGELOG.md), and section 9 there is the
migration procedure for a 2.0.1 memory database. SQLite remains the only fact
authority; host adapters share the same contracts.

**What is not verified.** `scripts/check.py --tier release` runs about 2,300 tests with
none failing. They check contracts, storage and the hosts' wiring; none of them
measures recall quality. Every accuracy figure in the notes was measured by us, on
our own corpora, by hand, and there is no regression suite you or we can re-run
automatically -- that is the first item in *What is not finished*.

## For agents: install or upgrade on the user's behalf

Users only need to ask "install Scope Recall" or "帮我升级一下 scoperecall".
Start with `scope-recall setup --host <hermes-or-codex> --home <actual-instance-home>`.
New users go directly to installation; only detected legacy databases go through
[the agent migration workflow](maintenance/AGENT_WORKFLOW.md). The installed
`scope-recall-setup` skill makes this routing discoverable in Hermes and Codex.
A second installed skill, `scope-recall-memory`, which Claude Code gets too, is for the owner's everyday questions:
what is remembered about me, did I say it or was it worked out, does it still hold, and
what correcting, muting or deleting one memory does before it is done.
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

The package is `hermes-scope-recall` on PyPI. Install it into the same isolated Python
environment the host uses:

```text
python -m pip install hermes-scope-recall==3.7.0
python -m pip install "hermes-scope-recall[codex]==3.7.0"
```

The same wheel and sdist are attached to the
[GitHub Release](https://github.com/410979729/scope-recall-hermes/releases/tag/v3.7.0)
alongside `SHA256SUMS` and `RELEASE-PROVENANCE.json`, for an offline install
(`python -m pip install "<path-to-wheel>"`). To build it yourself from this checkout instead:

```text
python -m build --wheel
```

Upgrading from 2.0.x is not an in-place upgrade. Read section 9 of
[CHANGELOG.md](CHANGELOG.md) before you start; your old database needs a
one-time offline migration and there are two errors people commonly hit.

The two console names `scope-recall` and `hermes-scope-recall` invoke the same v3 maintenance CLI. They are aliases for the current CLI only; neither is a compatibility promise for an older command set.

Use explicit absolute paths for installation planning. Hermes `--agent-id` must match the host active profile (`get_active_profile_name()`, commonly `default` on an isolated home). Hermes default `--agent-workspace` is `hermes` to match the host memory-provider init contract; pass the same value on plan and apply if you override it. Codex does not accept `--agent-workspace`.

If you talk to Hermes through the Desktop app or `hermes --tui` rather than the CLI, add `--local-platform desktop` (or `tui`) to both commands. Those surfaces name no user unless a dashboard login exists, and a session that names no user is refused everywhere but the CLI until the installer approves the surface; [docs/install.md](docs/install.md) says what the approval does and does not cover. If you reach the host through a dashboard login instead (`hermes serve` with `dashboard.basic_auth`, the Desktop app on another machine), the session is that login: approve it as yours with `--owner-login desktop=basic:<name>`. Hermes Desktop also builds
the Python environment it runs plugins in, and builds it again on updates: pass it
`--target-plugin-dir <home>\plugins\scope-recall`, where Hermes finds the plugin and the core it declares
even after a rebuild dropped the core (section 1 of [docs/install.md](docs/install.md)).

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

The `codex` extra adds the MCP SDK. Without that extra, the Core and Hermes paths remain importable. The `lancedb` extra enables the tested LanceDB companion path. PostgreSQL/pgvector is outside this v3 distribution; an explicit configuration error directs operators to retain the old installation and use the migration guide.

On Windows, choose a short data directory such as `C:\ScopeRecall\my-agent`. LanceDB appends index, table, and temporary file names to that path. If the resulting native path is too long, the worker reports `native_vector_path_too_long` before starting LanceDB or calling the embedding API. Use a short target directory for a new installation or an offline migration.

## Profile and entity read views

3.1.0 adds two shared read-only Core methods, `profile` and `entity`, and exposes them on both Hermes tools and Codex MCP. They return a deterministic categorized current-fact view, or an exact one-hop statement view, over admitted consolidated claims only. Raw chat is never silently turned into a profile. Incoming relations match the full scalar `value_text` only and keep recorded conditions and validity. Explicit project-name aliases may resolve when they are already admitted and still live; person aliases are not generalized. A `budget_tokens` value too small for even the minimal truthful envelope is a validation error, not an oversized view. See [docs/profile-entity.zh-CN.md](docs/profile-entity.zh-CN.md). An older installed wheel does not gain these tools until 3.1.0 is installed.

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
Hermes homes attach as entries, and from 3.3.0 so do Codex and Claude Code
(`attach --host codex|claude-code`). See [docs/shared-store.md](docs/shared-store.md).
WorkBuddy and dsh attach the same way (`attach --host workbuddy|dsh`).

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

The package root holds only the entry (`__init__.py`), the version and the protocol contracts. `core/` is the host-independent memory core over SQLite truth; `vector/` the rebuildable vector companions; `adapters/` the Hermes and Codex host adapters (Claude Code runs through the Codex one) and model transport; `runtime/` the background worker, budgets and scheduling; `maintenance/` install, doctor, upgrade and migration behind the operator CLI. Every shipped module is reachable by import from an entry point named in `packaging_hooks/module_inventory.py`; the wheel allowlist is derived from that, not typed.

## Development checks

The clean wheel must be tested outside the source checkout. At minimum, verify both CLI aliases, a read-only doctor result, Core capture and recall against a temporary installation, and the stdlib HTTP helper's bounded invalid-input response. Host registration, real gateway lifecycle, and production data are separate acceptance boundaries.

Historical release notes and the former v2 packaging contract remain available in the repository history at the `v2.0.1` tag. They are not current v3 usage instructions.
