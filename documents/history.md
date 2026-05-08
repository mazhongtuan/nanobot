# nanobot 会话历史（history）处理文档

## 1. 总体定位

`history` 在 nanobot 里指**单个会话（session）中尚未被归档的、按时间顺序排列的消息列表**，用于：

- 给 LLM 拼上下文（"history" 在 prompt 中位于 `system` 之后、当前输入之前）；
- 给上层做状态/容量判断；
- 给「总结归档」（memory consolidation）作为输入。

它和长期记忆（`memory/MEMORY.md`、`memory/HISTORY.md`）是**两套相互独立但有联系**的存储——history 是逐条原始消息（append-only，便于复用 LLM 缓存），长期记忆是被 LLM 总结后的可搜索文本。

---

## 2. 数据结构与持久化

### 2.1 `Session` 数据类

```28:33:D:\code\nanobot\nanobot\session\manager.py
    key: str  # channel:chat_id
    messages: list[dict[str, Any]] = field(default_factory=list)
    created_at: datetime = field(default_factory=datetime.now)
    updated_at: datetime = field(default_factory=datetime.now)
    metadata: dict[str, Any] = field(default_factory=dict)
    last_consolidated: int = 0  # Number of messages already consolidated to files
```

要点：

- `key` = `channel:chat_id`（特别的，subagent 写回时用 `system:<channel>:<chat_id>`，CLI 默认 `cli:direct`，心跳任务用 `heartbeat`）。
- `messages` 是**完整的、append-only** 的列表，不会被 in-place 修改。
- `last_consolidated` 是一个**前缀指针**：下标 `[0, last_consolidated)` 的消息已经被归档进 `MEMORY.md` / `HISTORY.md`，`get_history` 不再返回它们，但物理记录依旧保留在 `messages` 里。
- 每条 message 的字段形态：
  - `role`: `system` / `user` / `assistant` / `tool`
  - `content`: 文本字符串、或多模态块列表 `[{"type": "text"|"image_url", ...}]`
  - 助手轮可能带 `tool_calls`、`reasoning_content`、`thinking_blocks`（来自 `build_assistant_message`）
  - `tool` 角色带 `tool_call_id`、`name`
  - 持久化时由 `_save_turn` 注入 `timestamp`

### 2.2 文件格式

每个 session 一个 JSONL 文件，路径为 `<workspace>/sessions/<safe_key>.jsonl`：

```218:233:D:\code\nanobot\nanobot\session\manager.py
    def save(self, session: Session) -> None:
        """Save a session to disk."""
        path = self._get_session_path(session.key)

        with open(path, "w", encoding="utf-8") as f:
            metadata_line = {
                "_type": "metadata",
                ...
                "last_consolidated": session.last_consolidated
            }
            f.write(json.dumps(metadata_line, ensure_ascii=False) + "\n")
            for msg in session.messages:
                f.write(json.dumps(msg, ensure_ascii=False) + "\n")
```

- 第 1 行是元数据（含 `last_consolidated`），剩余每行是一条消息。
- 整文件是**全量覆写**（`open(..., "w")`），不是 append；append-only 是逻辑层面的，不是 I/O 层面的。
- `_load` 会优先读取 `<workspace>/sessions/`，否则尝试从 legacy 路径 `~/.nanobot/sessions/` 迁移过来。

### 2.3 `SessionManager` 缓存

```161:169:D:\code\nanobot\nanobot\session\manager.py
        if key in self._cache:
            return self._cache[key]

        session = self._load(key)
        if session is None:
            session = Session(key=key)

        self._cache[key] = session
        return session
```

进程内有 `_cache: dict[key, Session]`，所有 `get_or_create` 都走它；落盘后 `_cache[key] = session` 替换为最新。`invalidate(key)` 用于强制下次重新读盘（`/new` 命令会调用）。

---

## 3. 读取 history：`Session.get_history`

```69:93:D:\code\nanobot\nanobot\session\manager.py
    def get_history(self, max_messages: int = 500) -> list[dict[str, Any]]:
        """Return unconsolidated messages for LLM input, aligned to a legal tool-call boundary."""
        unconsolidated = self.messages[self.last_consolidated:]
        sliced = unconsolidated[-max_messages:]

        # Drop leading non-user messages to avoid starting mid-turn when possible.
        for i, message in enumerate(sliced):
            if message.get("role") == "user":
                sliced = sliced[i:]
                break

        # Some providers reject orphan tool results if the matching assistant
        # tool_calls message fell outside the fixed-size history window.
        start = self._find_legal_start(sliced)
        ...
        out: list[dict[str, Any]] = []
        for message in sliced:
            entry: dict[str, Any] = {"role": message["role"], "content": message.get("content", "")}
            for key in ("tool_calls", "tool_call_id", "name"):
                if key in message:
                    entry[key] = message[key]
            out.append(entry)
        return out
```

逐步规则：

1. **跳过已归档前缀**：起点是 `last_consolidated`；这之前的全部不参与 LLM。
2. **截取尾部 N 条**：`max_messages` 是「最多看多少条最近消息」；调用处普遍传 `0`（含义见下），等同于"全量未归档"，因为 Python 的负切片 `[-0:]` 实际等价于 `[len:]`，不过这里 `[-max_messages:]` 在 `max_messages == 0` 时 = `[0:]`，与"未截取"等价（参见后文）。
3. **左对齐到 user 边界**：从前向后丢弃所有非 `user` 的消息，避免 prompt 以 `tool` 或 `assistant` 开头导致提供方报错。
4. **校验 tool_call 配对**：`_find_legal_start` 扫描整段，凡发现"`tool` 消息引用的 `tool_call_id` 在前面没有对应的 `assistant.tool_calls` 声明"，就把切片起点抬到该 tool 消息之后。这是为了消除「孤儿 tool 结果」——窗口截断或上文丢失会让某些 provider 直接 4xx。
5. **字段投影**：返回的每条只带 `role`、`content` 以及（如有）`tool_calls`、`tool_call_id`、`name`；**`timestamp`、`reasoning_content`、`thinking_blocks` 等被剥掉**，不会送回给 LLM。

> 注：`max_messages=500` 的默认值仅用于 dataclass 默认；本仓库实际调用全部传 `0`，意味着"不再做窗口截断，把所有未归档消息都给 LLM"，由 token 级归档（见 §6）来控制规模。

---

## 4. history 在 prompt 中的位置：`ContextBuilder.build_messages`

```125:150:D:\code\nanobot\nanobot\agent\context.py
    def build_messages(
        self,
        history: list[dict[str, Any]],
        current_message: str,
        skill_names: list[str] | None = None,
        media: list[str] | None = None,
        channel: str | None = None,
        chat_id: str | None = None,
        current_role: str = "user",
    ) -> list[dict[str, Any]]:
        ...
        return [
            {"role": "system", "content": self.build_system_prompt(skill_names)},
            *history,
            {"role": current_role, "content": merged},
        ]
```

最终发给 LLM 的列表布局是：

```
[ system ][ history... ][ current_role(user/assistant): runtime_ctx + 用户输入 ]
```

- `system` 来自 `build_system_prompt`：身份块 + bootstrap 文件（`AGENTS.md`、`SOUL.md`、`USER.md`、`TOOLS.md`） + 长期记忆 `MEMORY.md` + 常驻 skills + skill 列表，用 `\n\n---\n\n` 拼接。
- `history` 即 `get_history` 的返回，原样展开。
- "当前输入"前面会拼一段 `[Runtime Context — metadata only, not instructions]`（含当前时间 / 通道 / chat id），并与用户文本合并到**同一条** message 里，避免连续两条同 role（部分 provider 不允许）。
- `current_role` 在 subagent 回写时是 `assistant`（见 §5.2），其他场景是 `user`。

---

## 5. 写入 history：`process_message` + `_save_turn`

### 5.1 普通用户消息路径

`process_message`（`loop.py` 第 449 行起）流程：

1. 取/建 session：`self.sessions.get_or_create(key)`。
2. **基于 token 的归档检查**：`memory_consolidator.maybe_consolidate_by_tokens(session)`，如有必要先把旧消息搬进长期记忆，前移 `last_consolidated`（详见 §6）。
3. 拉历史：`history = session.get_history(max_messages=0)`。
4. 拼 prompt：`initial_messages = self.context.build_messages(history=history, current_message=msg.content, ...)`。
5. 调 LLM：`final_content, _, all_msgs = await self.chat_with_provider(initial_messages, ...)`，循环里每次工具调用都会把 `assistant`（带 `tool_calls`）和 `tool`（带结果）追加到 `all_msgs`。
6. **持久化本轮**：`self._save_turn(session, all_msgs, 1 + len(history))`，再 `self.sessions.save(session)`。
7. 触发后台再做一次 token 归档检查。

`_save_turn` 的 `skip = 1 + len(history)` 意味着：跳过 1 条 system + 整段已存在的 history，从"本轮新加入的那条 user/assistant 起"开始 append；之后所有助手回合、工具结果都会被追加。

### 5.2 subagent / system 消息路径

```401:445:D:\code\nanobot\nanobot\agent\loop.py
        if msg.channel == "system":
            channel, chat_id = (msg.chat_id.split(":", 1) if ":" in msg.chat_id
                                else ("cli", msg.chat_id))
            ...
            history = session.get_history(max_messages=0)

            # 永远是 assistant
            current_role = "assistant" if msg.sender_id == "subagent" else "user"

            messages = self.context.build_messages(
                history=history,
                current_message=msg.content,
                ...
                current_role=current_role,
            )

            final_content, _, all_msgs = await self.chat_with_provider(...)

            self._save_turn(session, all_msgs, 1 + len(history))
            ...
```

子代理的回包以 `assistant` 角色注入父 session 的 history；写入逻辑与普通路径完全一致。

### 5.3 `_save_turn` 的清洗规则

```595:626:D:\code\nanobot\nanobot\agent\loop.py
    def _save_turn(self, session: Session, messages: list[dict], skip: int) -> None:
        """Save new-turn messages into session, truncating large tool results."""
        from datetime import datetime
        for m in messages[skip:]:
            entry = dict(m)
            role, content = entry.get("role"), entry.get("content")
            if role == "assistant" and not content and not entry.get("tool_calls"):
                continue  # skip empty assistant messages — they poison session context
            if role == "tool":
                if isinstance(content, str) and len(content) > self._TOOL_RESULT_MAX_CHARS:
                    entry["content"] = content[:self._TOOL_RESULT_MAX_CHARS] + "\n... (truncated)"
                elif isinstance(content, list):
                    filtered = self._sanitize_persisted_blocks(content, truncate_text=True)
                    if not filtered:
                        continue
                    entry["content"] = filtered
            elif role == "user":
                if isinstance(content, str) and content.startswith(ContextBuilder._RUNTIME_CONTEXT_TAG):
                    parts = content.split("\n\n", 1)
                    if len(parts) > 1 and parts[1].strip():
                        entry["content"] = parts[1]
                    else:
                        continue
                if isinstance(content, list):
                    filtered = self._sanitize_persisted_blocks(content, drop_runtime=True)
                    if not filtered:
                        continue
                    entry["content"] = filtered
            entry.setdefault("timestamp", datetime.now().isoformat())
            session.messages.append(entry)
        session.updated_at = datetime.now()
```

落盘前要做的修整（很重要）：

| 角色 | 处理 |
|------|------|
| `assistant`（空 content 且无 tool_calls） | **整条丢弃**——空助手消息会污染上下文 |
| `tool`（字符串内容） | 超过 `_TOOL_RESULT_MAX_CHARS = 16_000` 字符则截断并加 `... (truncated)` |
| `tool`（多模态块列表） | 走 `_sanitize_persisted_blocks(truncate_text=True)`：text 块按上限截断、`data:image/...` 大 base64 图被替换为 `[image: <path>]` 的占位 |
| `user`（字符串） | 若以 `_RUNTIME_CONTEXT_TAG` 开头，则**剥掉 runtime context 前缀**，只留真正用户文本；如果剥完为空就丢弃 |
| `user`（块列表） | `_sanitize_persisted_blocks(drop_runtime=True)`：删 runtime context 块、把内联 base64 图换成路径占位 |
| 其他 | 直接 append |
| 全部 | `setdefault("timestamp", now)` |

设计意图：**送给 LLM 的 prompt 与持久化到 history 的内容是不一样的**——多模态 base64、runtime metadata、超长工具输出都不应进 history（它们要么体积巨大，要么是一次性的环境信息），但 LLM 在那一轮看得到完整上下文。

### 5.4 助手轮的"脏"字段

`build_assistant_message` 会把 `reasoning_content` / `thinking_blocks` 写进 message 字典里，`_save_turn` 不主动剥，所以这些字段会落盘（占位空间）；但 `get_history` 投影只取 5 个键，下次组 prompt 时会自动忽略它们。这等价于"原文存档，但下一次不送给 LLM"。

---

## 6. 容量控制：基于 token 的归档（memory consolidation）

### 6.1 触发时机

`process_message` 的入口和出口各调一次：

- 入口：阻塞当前请求，确保即使 history 很大也能压缩到预算之内。
- 出口：通过 `_schedule_background` 异步补一次（写完日志再压）。

### 6.2 预算计算

```317:330:D:\code\nanobot\nanobot\agent\memory.py
            budget = self.context_window_tokens - self.max_completion_tokens - self._SAFETY_BUFFER
            target = budget // 2
            estimated, source = self.estimate_session_prompt_tokens(session)
            ...
            if estimated < budget:
                ...
                return
```

- `context_window_tokens`：构造 `AgentLoop` 时传入（默认 65536）。
- 留出 `max_completion_tokens` 给输出，`_SAFETY_BUFFER = 1024` 给分词器误差。
- 当估算 prompt > `budget` 时进入归档；目标是把 prompt 压到 ≤ `budget // 2`，留下足够余量避免抖动反复触发。

### 6.3 选边界

```258:278:D:\code\nanobot\nanobot\agent\memory.py
    def pick_consolidation_boundary(
        self,
        session: Session,
        tokens_to_remove: int,
    ) -> tuple[int, int] | None:
        """Pick a user-turn boundary that removes enough old prompt tokens."""
        start = session.last_consolidated
        ...
        for idx in range(start, len(session.messages)):
            message = session.messages[idx]
            if idx > start and message.get("role") == "user":
                last_boundary = (idx, removed_tokens)
                if removed_tokens >= tokens_to_remove:
                    return last_boundary
            removed_tokens += estimate_message_tokens(message)
```

- 归档块**必须以 user 消息为右边界**（即从 `[last_consolidated, idx)` 切走），保证剩余 history 仍然以 user 起头，不破坏 tool_call 配对。
- 找到的第一个能"够量"的 user 边界就是归档块；找不到则退化为最远的那一个 user 边界。

### 6.4 归档动作

```345:362:D:\code\nanobot\nanobot\agent\memory.py
                end_idx = boundary[0]
                chunk = session.messages[session.last_consolidated:end_idx]
                ...
                if not await self.consolidate_messages(chunk):
                    return
                session.last_consolidated = end_idx
                self.sessions.save(session)
```

- `consolidate_messages` 调用 `MemoryStore.consolidate`：让 LLM 通过 `save_memory` 工具产出 `history_entry`（追加到 `memory/HISTORY.md`，每条以 `[YYYY-MM-DD HH:MM]` 开头便于 grep）和 `memory_update`（覆盖 `memory/MEMORY.md`）。
- **不删消息**，只把 `last_consolidated` 前移；下一次 `get_history` 自然就跳过这些消息了。
- 如果连续三次归档失败（提供方拒绝、解析失败……），降级为 `_raw_archive`：直接把原始消息按 `[time] ROLE: content` 文本追加到 `HISTORY.md`，避免无限重试。
- 一次入口压缩最多循环 `_MAX_CONSOLIDATION_ROUNDS = 5` 次，仍超预算就放手让请求继续（依赖 provider 的容错）。

### 6.5 估算 token 用的"探针"

```280:295:D:\code\nanobot\nanobot\agent\memory.py
    def estimate_session_prompt_tokens(self, session: Session) -> tuple[int, str]:
        history = session.get_history(max_messages=0)
        ...
        probe_messages = self._build_messages(
            history=history,
            current_message="[token-probe]",
            channel=channel,
            chat_id=chat_id,
        )
        return estimate_prompt_tokens_chain(...)
```

注意它用了**真实的 `build_messages`** 来组装 prompt——所以系统提示、bootstrap 文件、长期记忆、skills、history、runtime 块全部计入估算。`estimate_prompt_tokens_chain` 优先使用 provider 自己的计数器（如有），否则 fallback 到 tiktoken。

---

## 7. 历史相关的命令与运维路径

### 7.1 `/new`：开新会话同时归档旧的

```69:82:D:\code\nanobot\nanobot\command\builtin.py
async def cmd_new(ctx: CommandContext) -> OutboundMessage:
    ...
    snapshot = session.messages[session.last_consolidated:]
    session.clear()
    loop.sessions.save(session)
    loop.sessions.invalidate(session.key)
    if snapshot:
        loop._schedule_background(loop.memory_consolidator.archive_messages(snapshot))
```

- 取出**还没归档的尾段**作 snapshot。
- `session.clear()` 把 messages 清空、`last_consolidated` 归零。
- 落盘后 invalidate 缓存；下一次进来会重新读盘（此时是空的）。
- 后台用 `archive_messages` 强保证持久化（最多 3 次失败后 raw 归档）。

### 7.2 `/status`：报告 history 规模

```55:65:D:\code\nanobot\nanobot\command\builtin.py
        content=build_status_content(
            ...
            session_msg_count=len(session.get_history(max_messages=0)),
            context_tokens_estimate=ctx_est,
        ),
```

`session_msg_count` 给的是"未归档消息条数"，`ctx_est` 用 §6.5 的探针估出真实 prompt token。

### 7.3 心跳任务的 history 上限

```653:657:D:\code\nanobot\nanobot\cli\commands.py
        # Keep a small tail of heartbeat history so the loop stays bounded
        # without losing all short-term context between runs.
        session = agent.sessions.get_or_create("heartbeat")
        session.retain_recent_legal_suffix(hb_cfg.keep_recent_messages)
        agent.sessions.save(session)
```

`retain_recent_legal_suffix(N)`：

- 保留尾部 N 条；
- 如果切点落在中途，向前回退到最近一个 user 消息；
- 再用 `_find_legal_start` 删掉前段任何孤儿 tool 结果；
- `last_consolidated` 同步减小（被丢弃的可能本来在已归档区，要把指针拉回来）。

这和 `get_history` 的边界规则是镜像关系：保留下来的尾段必然能直接当作合法的 LLM 输入。

---

## 8. 端到端数据流总览

```
                ┌──────────────────────┐
inbound msg ───►│ process_message      │
                │ ─ get_or_create      │
                │ ─ maybe_consolidate  │  (token 超预算 → 归档前缀)
                │ ─ get_history (slice │
                │   未归档 + 边界对齐) │
                │ ─ build_messages     │
                │   [system]+[hist]+[u]│
                │ ─ chat_with_provider │  (循环：assistant↔tool)
                │ ─ _save_turn(skip=   │  (清洗 → append 到 messages)
                │   1 + len(history))  │
                │ ─ sessions.save      │  (全量写 JSONL)
                │ ─ schedule bg consol │
                └──────────────────────┘
                              │
                              ▼
              workspace/sessions/<key>.jsonl  ←──── append-only 物理文件
                              │
                  last_consolidated 指针
                              │
                              ▼
              workspace/memory/HISTORY.md      ◄── grep 友好的归档
              workspace/memory/MEMORY.md       ◄── 长期记忆全量覆写
```

---

## 9. 关键不变量（理解和扩展时务必保留）

1. `session.messages` **永远只追加**，不修改、不重排、不删除（除 `/new` 显式 `clear()` 与心跳的 `retain_recent_legal_suffix`）。这是 LLM prompt 缓存生效的前提。
2. `get_history` 的输出永远满足：（a）以 `user` 开头；（b）任何 `tool` 都能在前面找到匹配的 `assistant.tool_calls`。
3. 落盘的内容 ≠ 送 LLM 的内容：`_save_turn` 会移除 runtime context、收缩工具大输出、占位 base64 图；`get_history` 投影会移除 `timestamp` / `reasoning_content` / `thinking_blocks`。
4. 容量控制只通过 `last_consolidated` 前移完成，不动 `messages`；这样可以重放、回看、修复。
5. session key 决定隔离粒度：`channel:chat_id`，子代理消息会被路由回**父会话**（`system:<channel>:<chat_id>` 解析后写到对应会话）。
