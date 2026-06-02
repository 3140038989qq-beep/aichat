# Agent Memory 系统

> 最后更新: 2026-06-02

## 启用方式

`config.json` 中设置：
```json
{
  "agent": true,
  "agent_workspace": "~/cow",
  "agent_max_context_tokens": 50000,
  "agent_max_context_turns": 20,
  "agent_max_steps": 20,
  "knowledge": true,
  "conversation_persistence": true
}
```

Agent 模式启用后，**消息通过 `channel.py:build_reply_content()` → `Bridge().fetch_agent_reply()` → `agent_bridge.py:agent_reply()`** 走完整 agent 链路。

## 记忆按用户隔离

### 改造前
所有用户的记忆混在一起：
```
~/cow/MEMORY.md
~/cow/memory/YYYY-MM-DD.md
```

### 改造后
每个微信用户独立目录：
```
~/cow/
├── MEMORY.md                              ← 全局默认（无用户ID时）
├── memory/
│   └── users/
│       └── {user_id}/
│           ├── MEMORY.md                  ← 该用户的长时记忆
│           ├── 2026-06-02.md             ← 该用户的每日记忆
│           └── dreams/
│               └── 2026-06-02.md         ← 该用户的梦日记
```

### 修改的 3 个代码路径

#### 1. `bridge/agent_bridge.py` — 注入 user_id
```python
agent._current_session_id = session_id
agent._current_user_id = session_id  # ← 新增
```

#### 2. `agent/prompt/workspace.py` — 加载 per-user MEMORY.md
```python
def load_context_files(workspace_dir, files_to_load=None, user_id=None):
    # ...加载全局 MEMORY.md
    # 额外加载 per-user MEMORY.md
    if user_id:
        user_memory_path = f"memory/users/{user_id}/MEMORY.md"
        # 读取并注入上下文
```

#### 3. `bridge/agent_initializer.py` — per-user Deep Dream
```python
# initialize_agent: 传入 session_id
context_files = load_context_files(workspace_root, user_id=session_id)

# _flush_all_agents: 每个用户独立 flush + dream
for label, agent in agents:
    user_id = getattr(agent, '_current_user_id', None)
    flush_mgr.create_daily_summary(messages, user_id=user_id)
# ...
for user_id, flush_mgr in dream_candidates:
    flush_mgr.deep_dream(user_id=user_id)
```

## 记忆生成时机

| 触发条件 | 做什么 | 频率 |
|---------|-------|------|
| 对话超过 20 轮 | 旧轮次裁剪 → LLM 摘要写入 `YYYY-MM-DD.md` | 每次超限自动 |
| Token 超限 | 紧急 flush 后清空上下文 | 故障时 |
| 每天 23:55 | Deep Dream: 当日记忆蒸馏到 `MEMORY.md` + 生成梦日记 | 每天一次 |

## 核心代码路径

```
用户消息
  → agent_bridge.py: agent_reply(query, context)
    → session_id = context["session_id"]   # 微信用户ID
    → agent = get_agent(session_id)
    → agent._current_user_id = session_id  # ← 注入
    → agent.run_stream(user_message)
      → agent_stream.py: _trim_messages()
        → flush_memory(messages, user_id=agent._current_user_id)
    → 回复用户

23:55 定时器
  → agent_initializer.py: _flush_all_agents()
    → 遍历所有 agent sessions
    → create_daily_summary(messages, user_id)
    → deep_dream(user_id)
```

## 相关文件索引

| 文件 | 用途 |
|------|------|
| `agent/memory/summarizer.py` | MemoryFlushManager, Deep Dream, LLM摘要 |
| `agent/memory/manager.py` | MemoryManager, 混合搜索 |
| `agent/memory/config.py` | MemoryConfig, 路径配置 |
| `agent/memory/conversation_store.py` | SQLite 对话持久化 |
| `agent/prompt/workspace.py` | 工作空间初始化, 上下文文件加载 |
| `agent/prompt/builder.py` | 系统提示词构建 (含记忆注入) |
| `agent/protocol/agent_stream.py` | 上下文裁剪 + flush 触发 |
| `bridge/agent_bridge.py` | AgentBridge, create_agent, agent_reply |
| `bridge/agent_initializer.py` | Agent 初始化, 记忆系统设置, 每日 flush |
| `channel/channel.py` | build_reply_content, Agent 模式判断 |
