---
title: "Hermes Agent v0.16 Kanban Swarm 功能解析"
date: 2026-08-26
tags: [Hermes, AI Agent, Kanban, 多智能体, 自动化]
author: azhegps
---

当你手里有一个需要多个角色并行协作才能完成的目标——比如"调研 + 写稿 + 校对 + 发布"——你的第一反应是什么？拆成若干独立任务，再手动协调它们的先后顺序和交接？听起来简单，做起来全是坑：卡片拆到一半忘了关联、某个 worker 卡死没人接手、协作过程事后完全无法审计。Hermes Agent v0.16 的 Kanban Swarm 就是冲着这些问题来的。

## 一条命令，生成一张协作拓扑图

传统的做法是层层 `delegate_task` 调用，父代理阻塞等待、子代理做完即走，中间状态进了上下文压缩就消失了。Kanban Swarm 换了个思路：把整个协作固化成一张**持久的依赖图**，落在 SQLite 里，每一张卡片都是可审计的一行。

```bash
hermes kanban swarm "写一篇 Hermes Kanban 深度解析文章" \
  --worker researcher:调研:grounded-citations \
  --worker writer:撰写 \
  --verifier reviewer --synthesizer publisher
```

执行后你会得到：1 张已完成的 root/blackboard 卡 + N 张并行的 worker 卡 + 1 张 gate 在所有 worker 上的 verifier 卡 + 1 张 gate 在 verifier 上的 synthesizer 卡。整张图**原子提交**——要么完整出现，要么完全不出现，dashboard 永远看不到"半链接"的残图。

**为什么重要**：手动拆卡时最常见的错误就是忘记画依赖边，导致下游在空输入上乱跑。Swarm 用一条命令把"谁等谁"全部编码进图里，你只负责描述目标，编排交给系统。

## 共享上下文：blackboard 以结构化 JSON 存于 root 卡

多角色协作的另一个痛点是**上下文怎么传递**。Swarm 的做法是把 blackboard（共享的事实与阶段结论）以结构化 JSON 评论的形式挂在 root 卡上，任何 worker 都能读写，上游 worker 的 `summary` + `metadata` 会原样注入下游 worker 的上下文。

**为什么重要**：`delegate_task` 的结果一旦返回父上下文就"熔断"了，重跑是全新的；而 Kanban 里每次握手都是留存的一行，重跑可以接着上次的结论走，跨重启也不会丢。

## 依赖图自动编排执行次序

worker 并行 → verifier 等**所有** worker 完成 → synthesizer 等 verifier 判为 clean 后才唤醒。整个执行次序完全由依赖图驱动，不需要任何手工编排脚本。

```bash
# 面向开发者的 worker lane 写法，支持指定 profile 与 skill
hermes kanban swarm <goal> --worker researcher:调研:skillA,skillB \
  --verifier reviewer --synthesizer writer
```

**为什么重要**：对比三类原语最能看清定位——`delegate_task` 是进程内的函数调用（fork→join，失败即失败）；cron 是时间驱动的闹钟（到期触发，无协作）；而 Kanban 是**持久消息队列 + 状态机**：崩溃可 reclaim、block 后人工 unblock 可重跑、任意点可人在环 comment。父代理只要一个短答案就选 `delegate_task`；跨 agent 边界、要跨重启存活、可能被不同角色接手、事后要可发现，就选 Kanban。

## 生命周期的健康终结器

每个 worker 的任务结束必须恰好调用三种终结器之一：`kanban_complete`（done）、`kanban_request_review`（进 review）、`kanban_block`（等人工）。一个也没调，dispatcher 会把它当作 crash 回收重跑。

**为什么重要**：这让"假装完成"无从遁形——worker 拿不出 `kanban_complete`，任务就永远留在 running，配合 `stranded_in_ready`（30 分钟无人认领即告警）和 `failure_limit`（连续失败自动 block），整条流水线的健康状况对操作者完全透明。

值得一提的是，本文从选题调研、撰写到校对，正是跑在一张真实的 Kanban Swarm 拓扑上——你正在读的这段文字，就是 synthesizer 依赖图触达的最终产物。
