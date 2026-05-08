数据流动概览

  一次实验的完整流程

  .env (API keys)
  configs/configs.yaml (LLM参数)
  tasks/configs.yaml (任务参数)
          │
          ▼
  tasks/run.py          ← 入口
    build_task()        ← 构建环境、记录器、任务列表
    build_mas()         ← 构建LLM、推理模块、记忆模块
    run_task()          ← 主循环

  ---
  单个任务的执行循环

  task_config (from data/*.jsonl)
          │
          ▼
  env.set_env()  →  task_main, task_description
          │
          ▼
  prompts/  →  task_instruction (system prompt + few-shots)
          │
          ▼
  memory.retrieve_memory(task_main)
    →  成功轨迹 + 失败轨迹 + insights
          │
          ▼
  mas.schedule(task_config)         ← MAS主循环
    │
    ├─ agents 互相通信，生成 action
    │    每步产生: AgentMessage → 存入 StateChain (nx.DiGraph)
    │
    ├─ env.step(action)  →  observation, reward, done
    │    MASMessage.move_state(action, observation)
    │
    └─ 循环直到 done 或超过 max_trials
          │
          ▼
  memory.add_memory(MASMessage)     ← 任务结束后写入记忆
    ├─ 稀疏化: 删除 reward < 0 的步骤
    ├─ ChromaDB: 存储轨迹文本 + 元数据
    ├─ TaskLayer (graph.pkl): 添加任务节点 + 相似度边
    └─ InsightsManager (insights.json): 定期 finetune/merge rules

  ---
  记忆的三层结构（G-Memory）

  查询时 (retrieve):
    query_task
        │
        ├─ TaskLayer.retrieve_related_task()
        │    相似度检索 → k-hop 邻居扩展 → 相关任务集合
        │
        ├─ ChromaDB.similarity_search()
        │    → 成功/失败轨迹 (MASMessage)
        │    → LLM 重排序 (importance_score)
        │
        └─ InsightsManager.query_insights_with_score()
             → 按任务相关性排序的 rules 列表

  写入时 (add_memory):
    每 rounds_per_insights 次 → finetune_insights() (LLM 增删改 rules)
    每 20 次               → merge_insights()    (FINCH 聚类 + 合并)

  ---
  关键数据对象

  ┌─────────────────┬─────────────────────────────────┬──────────────────────────────┐
  │      对象       │              含义               │            持久化            │
  ├─────────────────┼─────────────────────────────────┼──────────────────────────────┤
  │ MASMessage      │ 一个完整任务（描述+轨迹+label） │ ChromaDB                     │
  ├─────────────────┼─────────────────────────────────┼──────────────────────────────┤
  │ StateChain      │ 任务的逐步消息图序列            │ 序列化进 ChromaDB metadata   │
  ├─────────────────┼─────────────────────────────────┼──────────────────────────────┤
  │ AgentMessage    │ 单个 agent 的一次响应           │ 是 StateChain 的节点         │
  ├─────────────────┼─────────────────────────────────┼──────────────────────────────┤
  │ TaskLayer       │ 任务相似度图                    │ .db/.../task_layer_graph.pkl │
  ├─────────────────┼─────────────────────────────────┼──────────────────────────────┤
  │ InsightsManager │ 规则列表+分数                   │ .db/.../insights.json        │
  └─────────────────┴─────────────────────────────────┴──────────────────────────────┘