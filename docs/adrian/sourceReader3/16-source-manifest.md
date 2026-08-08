# 16. 源码完整清单

本清单用于核对阅读覆盖，不替代 [14-package-file-index.md](14-package-file-index.md) 中的职责说明。当前共列出 381 个生产 Go 文件；仓库另有 211 个 Go 测试文件，应按专题章节给出的测试关键词阅读。

## 1. 生产 Go 文件

### `internal/agent`

- `internal/agent/agent.go`
- `internal/agent/agent_tool.go`
- `internal/agent/agentic_fetch_tool.go`
### `internal/agent/agenttest`

- `internal/agent/agenttest/coordinator.go`
### `internal/agent`

- `internal/agent/aws_sso_refresh.go`
- `internal/agent/coordinator.go`
- `internal/agent/errors.go`
- `internal/agent/event.go`
- `internal/agent/hooked_tool.go`
### `internal/agent/hyper`

- `internal/agent/hyper/provider.go`
### `internal/agent`

- `internal/agent/loop_detection.go`
### `internal/agent/notify`

- `internal/agent/notify/notify.go`
### `internal/agent/prompt`

- `internal/agent/prompt/prompt.go`
### `internal/agent`

- `internal/agent/prompts.go`
- `internal/agent/run_marker.go`
- `internal/agent/runid.go`
### `internal/agent/tools`

- `internal/agent/tools/bash.go`
- `internal/agent/tools/crush_info.go`
- `internal/agent/tools/crush_logs.go`
- `internal/agent/tools/diagnostics.go`
- `internal/agent/tools/download.go`
- `internal/agent/tools/edit.go`
- `internal/agent/tools/edit_whitespace.go`
- `internal/agent/tools/fetch.go`
- `internal/agent/tools/fetch_helpers.go`
- `internal/agent/tools/fetch_types.go`
- `internal/agent/tools/glob.go`
- `internal/agent/tools/grep.go`
- `internal/agent/tools/job_kill.go`
- `internal/agent/tools/job_output.go`
- `internal/agent/tools/list_mcp_resources.go`
- `internal/agent/tools/ls.go`
- `internal/agent/tools/lsp_call_hierarchy.go`
- `internal/agent/tools/lsp_definition.go`
- `internal/agent/tools/lsp_helpers.go`
- `internal/agent/tools/lsp_rename.go`
- `internal/agent/tools/lsp_replace_symbol.go`
- `internal/agent/tools/lsp_restart.go`
- `internal/agent/tools/lsp_symbols.go`
- `internal/agent/tools/mcp-tools.go`
- `internal/agent/tools/mcp/channel.go`
- `internal/agent/tools/mcp/init.go`
- `internal/agent/tools/mcp/lifecycle.go`
- `internal/agent/tools/mcp/process_other.go`
- `internal/agent/tools/mcp/process_unix.go`
- `internal/agent/tools/mcp/prompts.go`
- `internal/agent/tools/mcp/resources.go`
- `internal/agent/tools/mcp/tools.go`
- `internal/agent/tools/multiedit.go`
- `internal/agent/tools/question.go`
- `internal/agent/tools/read_mcp_resource.go`
- `internal/agent/tools/references.go`
- `internal/agent/tools/rg.go`
- `internal/agent/tools/safe.go`
- `internal/agent/tools/search.go`
- `internal/agent/tools/sourcegraph.go`
- `internal/agent/tools/todos.go`
- `internal/agent/tools/tools.go`
- `internal/agent/tools/view.go`
- `internal/agent/tools/web_fetch.go`
- `internal/agent/tools/web_search.go`
- `internal/agent/tools/write.go`
### `internal/agent`

- `internal/agent/usage_fallback.go`
### `internal/ansiext`

- `internal/ansiext/ansi.go`
### `internal/app`

- `internal/app/app.go`
- `internal/app/lsp_events.go`
- `internal/app/provider.go`
- `internal/app/testing.go`
### `internal/backend`

- `internal/backend/agent.go`
- `internal/backend/backend.go`
- `internal/backend/config.go`
- `internal/backend/events.go`
- `internal/backend/filetracker.go`
- `internal/backend/permission.go`
- `internal/backend/question.go`
- `internal/backend/session.go`
- `internal/backend/testing.go`
- `internal/backend/util.go`
### `internal/client`

- `internal/client/client.go`
- `internal/client/config.go`
- `internal/client/dial_other.go`
- `internal/client/dial_windows.go`
- `internal/client/errors.go`
- `internal/client/proto.go`
### `internal/clipboard`

- `internal/clipboard/clipboard.go`
- `internal/clipboard/clipboard_not_supported.go`
- `internal/clipboard/clipboard_supported.go`
### `internal/cmd`

- `internal/cmd/dirs.go`
- `internal/cmd/login.go`
- `internal/cmd/logout.go`
- `internal/cmd/logs.go`
- `internal/cmd/models.go`
- `internal/cmd/projects.go`
- `internal/cmd/root.go`
- `internal/cmd/root_other.go`
- `internal/cmd/root_windows.go`
- `internal/cmd/run.go`
- `internal/cmd/schema.go`
- `internal/cmd/server.go`
- `internal/cmd/server_other.go`
- `internal/cmd/server_windows.go`
- `internal/cmd/session.go`
- `internal/cmd/stats.go`
- `internal/cmd/update_providers.go`
### `internal/commands`

- `internal/commands/commands.go`
### `internal/config`

- `internal/config/atomicwrite.go`
- `internal/config/atomicwrite_unix.go`
- `internal/config/atomicwrite_windows.go`
- `internal/config/catwalk.go`
- `internal/config/config.go`
- `internal/config/config_unix.go`
- `internal/config/config_windows.go`
- `internal/config/copilot.go`
- `internal/config/docker_mcp.go`
- `internal/config/hyper.go`
- `internal/config/init.go`
- `internal/config/load.go`
- `internal/config/provider.go`
- `internal/config/resolve.go`
- `internal/config/scope.go`
- `internal/config/store.go`
### `internal/csync`

- `internal/csync/doc.go`
- `internal/csync/maps.go`
- `internal/csync/slices.go`
- `internal/csync/value.go`
- `internal/csync/versionedmap.go`
### `internal/db`

- `internal/db/connect.go`
- `internal/db/connect_modernc.go`
- `internal/db/connect_ncruces.go`
- `internal/db/datadirlock.go`
- `internal/db/db.go`
- `internal/db/files.sql.go`
- `internal/db/messages.sql.go`
- `internal/db/models.go`
- `internal/db/querier.go`
- `internal/db/read_files.sql.go`
- `internal/db/sessions.sql.go`
- `internal/db/stats.sql.go`
### `internal/diff`

- `internal/diff/diff.go`
### `internal/diffdetect`

- `internal/diffdetect/detect.go`
### `internal/discover`

- `internal/discover/discover.go`
- `internal/discover/enricher.go`
- `internal/discover/litellm.go`
- `internal/discover/llamacpp.go`
- `internal/discover/lmstudio.go`
- `internal/discover/ollama.go`
- `internal/discover/omlx.go`
### `internal/dns`

- `internal/dns/android.go`
### `internal/env`

- `internal/env/env.go`
### `internal/event`

- `internal/event/all.go`
- `internal/event/event.go`
- `internal/event/identifier.go`
- `internal/event/logger.go`
### `internal/filepathext`

- `internal/filepathext/filepath.go`
### `internal/filetracker`

- `internal/filetracker/service.go`
### `internal/format`

- `internal/format/spinner.go`
### `internal/fsext`

- `internal/fsext/drive_other.go`
- `internal/fsext/drive_windows.go`
- `internal/fsext/expand.go`
- `internal/fsext/fileutil.go`
- `internal/fsext/lookup.go`
- `internal/fsext/ls.go`
- `internal/fsext/owner_others.go`
- `internal/fsext/owner_windows.go`
- `internal/fsext/paste.go`
### `internal/herdr`

- `internal/herdr/client.go`
- `internal/herdr/translate.go`
### `internal/history`

- `internal/history/file.go`
### `internal/home`

- `internal/home/home.go`
### `internal/hooks`

- `internal/hooks/hooks.go`
- `internal/hooks/input.go`
- `internal/hooks/runner.go`
### `internal/lock`

- `internal/lock/lock.go`
- `internal/lock/lock_unix.go`
- `internal/lock/lock_windows.go`
### `internal/log`

- `internal/log/http.go`
- `internal/log/log.go`
### `internal/lsp`

- `internal/lsp/client.go`
- `internal/lsp/handlers.go`
- `internal/lsp/manager.go`
### `internal/lsp/util`

- `internal/lsp/util/edit.go`
### `internal/message`

- `internal/message/attachment.go`
- `internal/message/content.go`
- `internal/message/message.go`
### `internal/oauth/callback`

- `internal/oauth/callback/page.go`
### `internal/oauth/copilot`

- `internal/oauth/copilot/client.go`
- `internal/oauth/copilot/disk.go`
- `internal/oauth/copilot/http.go`
- `internal/oauth/copilot/oauth.go`
- `internal/oauth/copilot/urls.go`
### `internal/oauth/hyper`

- `internal/oauth/hyper/device.go`
### `internal/oauth/mcp`

- `internal/oauth/mcp/handler.go`
- `internal/oauth/mcp/savingtokensource.go`
### `internal/oauth`

- `internal/oauth/token.go`
### `internal/permission`

- `internal/permission/permission.go`
### `internal/projects`

- `internal/projects/projects.go`
### `internal/proto`

- `internal/proto/agent.go`
- `internal/proto/history.go`
- `internal/proto/mcp.go`
- `internal/proto/message.go`
- `internal/proto/permission.go`
- `internal/proto/proto.go`
- `internal/proto/requests.go`
- `internal/proto/server.go`
- `internal/proto/session.go`
- `internal/proto/skills.go`
- `internal/proto/tools.go`
- `internal/proto/version.go`
### `internal/pubsub`

- `internal/pubsub/broker.go`
- `internal/pubsub/events.go`
### `internal/question`

- `internal/question/question.go`
### `internal/server`

- `internal/server/config.go`
- `internal/server/events.go`
- `internal/server/logging.go`
- `internal/server/net_other.go`
- `internal/server/net_windows.go`
- `internal/server/proto.go`
- `internal/server/recover.go`
- `internal/server/server.go`
- `internal/server/socket.go`
- `internal/server/socket_classify.go`
### `internal/session`

- `internal/session/session.go`
### `internal/shell`

- `internal/shell/background.go`
- `internal/shell/builtins_registry.go`
- `internal/shell/coreutils.go`
- `internal/shell/coreutils_exec.go`
- `internal/shell/coreutils_exec_stub.go`
- `internal/shell/dispatch.go`
- `internal/shell/doc.go`
- `internal/shell/exec_unix.go`
- `internal/shell/exec_windows.go`
- `internal/shell/expand.go`
- `internal/shell/jq.go`
- `internal/shell/persist_message.go`
- `internal/shell/run.go`
- `internal/shell/shell.go`
- `internal/shell/stream.go`
### `internal/shellconfig`

- `internal/shellconfig/builder.go`
- `internal/shellconfig/flags.go`
- `internal/shellconfig/hook.go`
- `internal/shellconfig/load.go`
- `internal/shellconfig/lsp.go`
- `internal/shellconfig/mcp.go`
- `internal/shellconfig/model.go`
- `internal/shellconfig/options.go`
- `internal/shellconfig/permissions.go`
- `internal/shellconfig/provider.go`
- `internal/shellconfig/register.go`
### `internal/skills`

- `internal/skills/catalog.go`
- `internal/skills/embed.go`
- `internal/skills/manager.go`
- `internal/skills/skills.go`
- `internal/skills/tracker.go`
### `internal/stringext`

- `internal/stringext/string.go`
### `internal/swagger`

- `internal/swagger/docs.go`
### `internal/ui/anim`

- `internal/ui/anim/anim.go`
### `internal/ui/attachments`

- `internal/ui/attachments/attachments.go`
### `internal/ui/chat`

- `internal/ui/chat/agent.go`
- `internal/ui/chat/assistant.go`
- `internal/ui/chat/bash.go`
- `internal/ui/chat/call_hierarchy.go`
- `internal/ui/chat/definition.go`
- `internal/ui/chat/diagnostics.go`
- `internal/ui/chat/docker_mcp.go`
- `internal/ui/chat/fetch.go`
- `internal/ui/chat/file.go`
- `internal/ui/chat/generic.go`
- `internal/ui/chat/lsp_restart.go`
- `internal/ui/chat/mcp.go`
- `internal/ui/chat/messages.go`
- `internal/ui/chat/question.go`
- `internal/ui/chat/references.go`
- `internal/ui/chat/rename.go`
- `internal/ui/chat/replace_symbol.go`
- `internal/ui/chat/search.go`
- `internal/ui/chat/shell.go`
- `internal/ui/chat/streaming_markdown.go`
- `internal/ui/chat/symbols.go`
- `internal/ui/chat/todos.go`
- `internal/ui/chat/tool_result_content.go`
- `internal/ui/chat/tools.go`
- `internal/ui/chat/unified_diff.go`
- `internal/ui/chat/user.go`
### `internal/ui/common`

- `internal/ui/common/ansi16.go`
- `internal/ui/common/button.go`
- `internal/ui/common/capabilities.go`
- `internal/ui/common/chromastyle.go`
- `internal/ui/common/common.go`
- `internal/ui/common/diff.go`
- `internal/ui/common/elements.go`
- `internal/ui/common/highlight.go`
- `internal/ui/common/interface.go`
- `internal/ui/common/markdown.go`
- `internal/ui/common/scrollbar.go`
- `internal/ui/common/timer.go`
### `internal/ui/completions`

- `internal/ui/completions/completions.go`
- `internal/ui/completions/item.go`
- `internal/ui/completions/keys.go`
### `internal/ui/dialog`

- `internal/ui/dialog/actions.go`
- `internal/ui/dialog/api_key_input.go`
- `internal/ui/dialog/arguments.go`
- `internal/ui/dialog/aws_sso.go`
- `internal/ui/dialog/commands.go`
- `internal/ui/dialog/commands_item.go`
- `internal/ui/dialog/common.go`
- `internal/ui/dialog/dialog.go`
- `internal/ui/dialog/filepicker.go`
- `internal/ui/dialog/inline_editor.go`
- `internal/ui/dialog/mcp_auth.go`
- `internal/ui/dialog/models.go`
- `internal/ui/dialog/models_item.go`
- `internal/ui/dialog/models_list.go`
- `internal/ui/dialog/notifications.go`
- `internal/ui/dialog/oauth.go`
- `internal/ui/dialog/oauth_copilot.go`
- `internal/ui/dialog/oauth_hyper.go`
- `internal/ui/dialog/permissions.go`
- `internal/ui/dialog/question_choice_base.go`
- `internal/ui/dialog/question_confirm.go`
- `internal/ui/dialog/question_editor.go`
- `internal/ui/dialog/question_form.go`
- `internal/ui/dialog/question_freetext.go`
- `internal/ui/dialog/question_multi.go`
- `internal/ui/dialog/question_single.go`
- `internal/ui/dialog/question_yesno.go`
- `internal/ui/dialog/quit.go`
- `internal/ui/dialog/reasoning.go`
- `internal/ui/dialog/sessions.go`
- `internal/ui/dialog/sessions_item.go`
### `internal/ui/diffview`

- `internal/ui/diffview/diffview.go`
- `internal/ui/diffview/split.go`
- `internal/ui/diffview/style.go`
- `internal/ui/diffview/util.go`
### `internal/ui/image`

- `internal/ui/image/image.go`
### `internal/ui/list`

- `internal/ui/list/filterable.go`
- `internal/ui/list/focus.go`
- `internal/ui/list/highlight.go`
- `internal/ui/list/item.go`
- `internal/ui/list/list.go`
### `internal/ui/logo`

- `internal/ui/logo/example/main.go`
- `internal/ui/logo/letterforms.go`
- `internal/ui/logo/logo.go`
- `internal/ui/logo/rand.go`
### `internal/ui/model`

- `internal/ui/model/chat.go`
- `internal/ui/model/filter.go`
- `internal/ui/model/header.go`
- `internal/ui/model/history.go`
- `internal/ui/model/keys.go`
- `internal/ui/model/landing.go`
- `internal/ui/model/lsp.go`
- `internal/ui/model/mcp.go`
- `internal/ui/model/mcp_auth.go`
- `internal/ui/model/onboarding.go`
- `internal/ui/model/pills.go`
- `internal/ui/model/session.go`
- `internal/ui/model/sidebar.go`
- `internal/ui/model/skills.go`
- `internal/ui/model/status.go`
- `internal/ui/model/ui.go`
- `internal/ui/model/workspace_cache.go`
### `internal/ui/notification`

- `internal/ui/notification/bell.go`
- `internal/ui/notification/icon_darwin.go`
- `internal/ui/notification/icon_other.go`
- `internal/ui/notification/native.go`
- `internal/ui/notification/native_beeep.go`
- `internal/ui/notification/native_stub.go`
- `internal/ui/notification/noop.go`
- `internal/ui/notification/notification.go`
- `internal/ui/notification/osc.go`
### `internal/ui/styles`

- `internal/ui/styles/grad.go`
- `internal/ui/styles/quickstyle.go`
- `internal/ui/styles/styles.go`
- `internal/ui/styles/themes.go`
### `internal/ui/util`

- `internal/ui/util/util.go`
### `internal/ui/xchroma`

- `internal/ui/xchroma/chroma.go`
### `internal/update`

- `internal/update/update.go`
### `internal/version`

- `internal/version/version.go`
### `internal/workspace`

- `internal/workspace/app_workspace.go`
- `internal/workspace/client_workspace.go`
- `internal/workspace/workspace.go`
### `根目录`

- `main.go`

## 2. 非 Go 行为源

下面这些 SQL、迁移、Prompt、工具说明和 Skill 也直接决定运行行为，不能因为不是 Go 而跳过。

- `internal/agent/templates/agent_tool.md`
- `internal/agent/templates/agentic_fetch.md`
- `internal/agent/templates/agentic_fetch_prompt.md.tpl`
- `internal/agent/templates/coder.md.tpl`
- `internal/agent/templates/initialize.md.tpl`
- `internal/agent/templates/summary.md`
- `internal/agent/templates/task.md.tpl`
- `internal/agent/templates/title.md`
- `internal/agent/tools/bash.md.tpl`
- `internal/agent/tools/crush_info.md`
- `internal/agent/tools/crush_logs.md.tpl`
- `internal/agent/tools/diagnostics.md`
- `internal/agent/tools/download.md.tpl`
- `internal/agent/tools/edit.md`
- `internal/agent/tools/fetch.md.tpl`
- `internal/agent/tools/glob.md.tpl`
- `internal/agent/tools/grep.md.tpl`
- `internal/agent/tools/job_kill.md`
- `internal/agent/tools/job_output.md`
- `internal/agent/tools/list_mcp_resources.md`
- `internal/agent/tools/ls.md.tpl`
- `internal/agent/tools/lsp_call_hierarchy.md`
- `internal/agent/tools/lsp_definition.md`
- `internal/agent/tools/lsp_rename.md`
- `internal/agent/tools/lsp_replace_symbol.md`
- `internal/agent/tools/lsp_restart.md`
- `internal/agent/tools/lsp_symbols.md`
- `internal/agent/tools/multiedit.md`
- `internal/agent/tools/question.md`
- `internal/agent/tools/read_mcp_resource.md`
- `internal/agent/tools/references.md`
- `internal/agent/tools/sourcegraph.md.tpl`
- `internal/agent/tools/todos.md`
- `internal/agent/tools/view.md.tpl`
- `internal/agent/tools/web_fetch.md.tpl`
- `internal/agent/tools/web_search.md.tpl`
- `internal/agent/tools/write.md`
- `internal/cmd/stats/AGENTS.md`
- `internal/db/migrations/20250424200609_initial.sql`
- `internal/db/migrations/20250515105448_add_summary_message_id.sql`
- `internal/db/migrations/20250624000000_add_created_at_indexes.sql`
- `internal/db/migrations/20250627000000_add_provider_to_messages.sql`
- `internal/db/migrations/20250810000000_add_is_summary_message.sql`
- `internal/db/migrations/20250812000000_add_todos_to_sessions.sql`
- `internal/db/migrations/20260127000000_add_read_files_table.sql`
- `internal/db/sql/files.sql`
- `internal/db/sql/messages.sql`
- `internal/db/sql/read_files.sql`
- `internal/db/sql/sessions.sql`
- `internal/db/sql/stats.sql`
- `internal/oauth/callback/AGENTS.md`
- `internal/skills/builtin/crush-config/SKILL.md`
- `internal/skills/builtin/crush-hooks/SKILL.md`
- `internal/skills/builtin/jq/SKILL.md`
- `internal/ui/AGENTS.md`

## 3. 生成代码识别

- `internal/db/*.sql.go` 与 `querier.go` 由 sqlc 生成，事实来源是 `internal/db/sql/*.sql` 与 migrations。
- `internal/swagger/docs.go` 是 Swagger 生成物，事实来源是 Server 路由与注释。
- 平台后缀文件不是重复代码；Go 会按目标平台选择其中一组。
- 测试辅助和 golden 虽不进入生产二进制，但记录并发、协议和视觉不变量。
