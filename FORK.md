# Fork 说明（个人维护）

上游：`github.com/looplj/axonhub`
本 fork：`github.com/B1anYu/axonhub`（public），本地 `~/dev/4_oss/axonhub`
基线：tag `v1.0.0-beta10`（与部署中的镜像同版本，不夹带升级）
分支：`fork/gemini-cache-fix`

> 部署/运维上下文在主栈文档 `~/ai-stack/docs/07-axonhub-upgrade.md` 10.4；
> 本文件只放「这个 fork 本身」的上下文。`AGENTS.md` / `CLAUDE.md` 是上游文件，
> 不要往里面写东西（rebase 必冲突）。

## 1. 为什么二开

Claude Code 2.1.3xx 会在对话中途往 `messages` 里追加真正的 `role: "system"` 消息：

- `# Environment update`（工作目录变化 / 文件被外部改动）
- `Note: /xxx changed on disk ...`
- `[SYSTEM NOTIFICATION - NOT USER INPUT] ...`（后台任务事件）
- `Contents of .../CLAUDE.md: ...`（新读的规则文件）

axonhub 的 Gemini 出站转换器（`llm/transformer/gemini/outbound_convert.go`，
`case "system", "developer"`）会把**所有** system 消息按顺序追加进
`systemInstruction`。Gemini 的隐式缓存要求 `systemInstruction` 完全静态，
所以每追加一条，整段前缀失效 → `cachedContentTokenCount=0`，并且 Gemini 的
隐式缓存是 best-effort，要 3~8 轮才重新预热。

2026-11-13 实测（ch13，studioarch，gemini-4-carbon-high）：

| 情况 | 次数 | 结果 |
|---|---|---|
| `systemInstruction.parts` 严格追加 | 10 | 10 次 cached=0 |
| `systemInstruction` 不变 | 126 | 126 次命中 |

MISS 平均 42.6s / 首字 31.6s；HIT 25.7s / 17.0s。每次变更约 3~8 轮
40 万 token 全量 prefill。表现上像「某个 session 没缓存」，实际只取决于
该 session 是否刚产生了一条会被前移的 system 消息。

## 2. 补丁

上游其实已经实现了正确做法：`llm/pipeline/cc/system_messages.go` 的
`SystemCacheCompatibility`（PR #2172）会把「首段连续 system 块之后」的
system 消息**原地降级成 `user`**，保留顺序与内容。但
`supportsSystemCacheCompatibility()` 的白名单只有：

```
openai chat / responses / responses compact / anthropic messages
```

漏了 `gemini/contents`。本 fork 只做两处修改：

1. `llm/pipeline/cc/system_messages.go`：白名单加入 `llm.APIFormatGeminiContents`
2. `llm/pipeline/cc/system_messages_test.go`：把 gemini 加入 affected 列表，
   原来的 "unaffected outbound protocol" 用例换成 `APIFormatGeminiEmbedding`

补丁后的效果：`systemInstruction` 只保留开头连续的 system 块（Claude Code
的 2~3 段），中途 system 消息原位置落入 `contents`，`role=user`，前缀稳定。

**如果上游以后把 `gemini/contents` 补进白名单，本 fork 即可废弃**（rebase 时
若发现白名单已包含，直接丢这个 commit）。

## 3. 项目结构注意点

- `llm/` 是**独立 Go module**（有自己的 `go.mod`），测试要
  `cd llm && go test ./pipeline/cc/...`，在根目录跑不到
- 根的 `go.mod` 通过 replace/require 依赖它；Dockerfile 也单独 copy 两个 module
- 前端在 `frontend/`（pnpm），Dockerfile 里先构建前端再编译 Go
- 版本号来自 `internal/build/VERSION`，CI 在构建前写入

## 4. 构建与发布

`.github/workflows/fork-image.yml`：

- 触发：push 到 `fork/**`，或手动 dispatch；只改 `**.md` 不触发
- 先跑 `llm/pipeline/cc` 单测，再 buildx 构建 `linux/amd64`
- 推送 `ghcr.io/b1anyu/axonhub:dev` 和 `:sha-<commit>`；GHCR 包为 public，
  匿名可拉（`docker manifest inspect` 可验证）
- Actions 缓存用 `type=gha`，二次构建约 2~5 分钟

## 5. 验证补丁是否生效

探针请求（UA 必须是 `claude-cli/...`，中间件靠它识别；否则不会降级）：

```json
{
  "model": "gemini-4-carbon-high",
  "max_tokens": 32,
  "stream": false,
  "system": [{"type":"text","text":"You are a test harness. Reply with one word."}],
  "messages": [
    {"role":"user","content":"hi"},
    {"role":"assistant","content":"hello"},
    {"role":"system","content":"# Environment update\n - Primary working directory: /tmp/probe"},
    {"role":"user","content":"say ok"}
  ]
}
```

发完看 body-storage 里该执行的出站 body（`requests/<id>/executions/<eid>/request_body.json`）：

- `systemInstruction.parts` 只有 1 段（只有开头 system）
- `contents` 里出现该 system 消息，且 `role=user`

## 6. 预检新镜像（换 tag / rebase 后）

1. 用 `sqlite3.Connection.backup()` 复制线上库到 `/tmp/axh-test/`
2. 临时容器 `127.0.0.1:8091`，挂库副本 + 同款 `config.yml`，起新镜像
3. 确认 `/health` 200、日志版本号正确、渠道加载数量正常
4. `docker rm -f` 并删目录

## 7. 同步上游

```bash
git fetch origin --tags
git checkout fork/gemini-cache-fix
git rebase v1.0.0-betaXX
git push --force-with-lease   # 触发 Actions 重新出 :dev
```

冲突只会出现在 `system_messages.go` 及其测试。升级前**必须备份 db**；
SQLite 迁移单向，回滚镜像 ≠ 回滚数据。本 fork 未引入任何 schema 迁移。

## 8. 已知遗留

- 缓存预热仍然是 best-effort：长时间空闲（>5 分钟）或上游侧淘汰后，第一两轮
  仍可能 miss，这是 Gemini 隐式缓存本身的行为，不是本补丁的问题
- 渠道 13 上原有的 `array_remove` 覆写（删 `<total_tokens>BUDGET...`）已无匹配
  对象，属无害死配置
- 全局 `prompt_protection_rules` 的 budget 替换已在 2026-11-13 删除，补丁生效后
  不再需要
