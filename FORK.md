# Fork 说明（个人维护）

基线：上游 tag `v1.0.0-beta10`（与部署中的镜像同版本）。
分支：`fork/gemini-cache-fix`。

## 补丁内容

`llm/pipeline/cc/system_messages.go`：`supportsSystemCacheCompatibility()` 的白名单加入 `llm.APIFormatGeminiContents`。

## 为什么

- Claude Code 2.1.3xx 会在对话中途追加真正的 `role: "system"` 消息（`# Environment update`、`Note: ... changed on disk`、`[SYSTEM NOTIFICATION]`、新读的 `CLAUDE.md` 内容等）。
- axonhub 的 Gemini 出站转换器（`llm/transformer/gemini/outbound_convert.go`）会把所有 `system`/`developer` 消息**按顺序追加进 `systemInstruction`**。
- 每追加一条，`systemInstruction` 就变一次 → Gemini 隐式缓存的整段前缀失效（`cachedContentTokenCount=0`），需要 3~8 轮才能重新预热。
- 上游已有中间件 `cc.SystemCacheCompatibility` 专门把「首段连续 system 块之后」的 system 消息降级成 `user`（原地保留，不改顺序），但白名单漏了 `gemini/contents`。本 fork 只补这一项。

补上后，动态 system 消息落入 `contents` 尾部，前缀保持稳定；**不再需要用全局 prompt-protection 规则做正则替换**（渠道级 `bodyOverrideOperations` 也不需要，可保留也可清理）。

## 构建

push 到 `fork/**` 或手动 `workflow_dispatch` 触发 `.github/workflows/fork-image.yml`：

- 先跑 `llm/pipeline/cc` 单元测试
- 再构建 `linux/amd64` 镜像并推：
  - `ghcr.io/b1anyu/axonhub:dev`
  - `ghcr.io/b1anyu/axonhub:sha-<commit>`

## 部署

`~/ai-stack/axonhub/docker-compose.yml` 中 image 改为 `ghcr.io/b1anyu/axonhub:dev`，然后：

```bash
docker compose up -d   # 会重建容器，必须由用户操作（会掐断当前 AI 会话）
```

⚠️ 不要用 `docker restart`（读的是旧镜像）。

## 跟踪上游

```bash
git fetch origin --tags
git checkout fork/gemini-cache-fix
git rebase v1.0.0-betaXX   # 新 tag
git push --force-with-lease
```

冲突点只会在 `llm/pipeline/cc/system_messages.go` 及其测试，一般一行。

## 验证

逐个请求看上游返回的 `usageMetadata.cachedContentTokenCount`：`systemInstruction` 段数稳定时应当接近 `promptTokenCount`；出现新的 `# Environment update` 时也不应再归零。
