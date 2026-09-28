# 05-Agent协调器与会话管理.md 证据索引

## 源码范围
- internal/agent/coordinator.go (1877 lines)
- internal/agent/agent.go (2393 lines)
- internal/agent/channel.go (18 lines)
- internal/agent/channelreply.go (307 lines)
- internal/agent/hyper_credits.go (103 lines)
- internal/agent/request_timeout.go (123 lines)
- internal/agent/loop_detection.go (92 lines)
- internal/agent/prompts.go (53 lines)
- internal/agent/templates/plan.md.tpl

## 关键变更（自 47b11b84 以来）

### 新增文件
1. internal/agent/channel.go - WithChannel, ChannelFromContext
2. internal/agent/channelreply.go - resolveChannelReply, sendChannelReply, discoverChannelReply
3. internal/agent/hyper_credits.go - hyperCreditsModel, newHyperCreditsModel, refreshBalance
4. internal/agent/request_timeout.go - requestTimeoutModel, newRequestTimeoutModel, requestTimeoutError
5. internal/agent/loop_detection.go - hasRepeatedToolCalls, getToolInteractionSignature
6. internal/agent/templates/plan.md.tpl - plan mode system prompt

### 关键修改
1. coordinator.go: NewCoordinator 新增 plan agent 构建 (L239-253)
2. coordinator.go: buildAgentModels 新增 requestTimeout + hyperCredits 包装 (L968-1070)
3. coordinator.go: run 新增 WaitForInitBudget, syncSessionChannel, ChannelFromContext (L303-427)
4. coordinator.go: effectiveReasoningEffort 新增 (L463-478)
5. coordinator.go: copilotResponsesModels 新增 gpt-6-astra, grok-4.5, grok-4.6 (L75-89)
6. coordinator.go: isOpenCodeMessagesModel/isOpenCodeResponsesModel 新增 (L94-117)
7. coordinator.go: syncSessionChannel 新增 (L440-458)
8. agent.go: updateSessionUsage 新增 CacheCreationTokens (L1915, L2007)
9. agent.go: getSessionMessages 使用 ListFromSummary (L1760-1774)
10. agent.go: filterOrphanedToolResults, syntheticToolResultsForOrphanedCalls 新增 (L1640-1746)

## 文档更新计划
- §1 buildAgentModels: 补充 requestTimeout + hyperCredits 包装管道
- §2 run: 补充 WaitForInitBudget, syncSessionChannel, ChannelFromContext
- §2 run: 补充 effectiveReasoningEffort, top_k, enable_thinking 处理
- §6 OnToolResult: 补充 orphaned/synthetic results, bash spill files
- §7 OnStepFinish: 补充 cache-creation tokens
- §10 getSessionMessages: 补充 ListFromSummary
- §12 Sub-Agent: 补充 plan agent
- §13 UpdateModels: 补充 hyper credits + timeout wrappers
- 新增 §15: Plan Mode
- 新增 §16: Claude Channels
- 新增 §17: Request Timeout
- 新增 §18: Loop Detection
