# mcp-registry-publish

谦面（facet）公开 MCP server → 官方 MCP Registry 的**发布管道**。

只有管道，没有应用代码：脚本从生产 facet 服务拉取 `/api/mcp-registry.json`
投影，逐条交给 `mcp-publisher publish` 上架到
`registry.modelcontextprotocol.io`，命名空间 `io.github.FindDataTechnology/*`。

## 为什么单独一个仓

`workflow_dispatch` 要求 workflow 文件在**默认分支**上；而主源码仓
`fd-craw-private` 是 **PRIVATE**，其账号私有 Actions 额度已耗尽（2011/2000），
run 会在 5 秒内红。公开仓标准 runner **免费无限**，且 OIDC 按
**仓库所属组织**授权命名空间——所以一个只放管道的公开仓是这件事最省的正确落点。

## 为什么必须 OIDC

交互式 `mcp-publisher login github` 走设备流，拿到的是 GitHub App token，
**读不到组织成员关系** → registry 只授予个人 `io.github.<username>/*`
（上游 Warning + registry issue #1468）。发组织名下只有两条路：

1. **GitHub Actions OIDC**（本仓，免费、无凭据落盘）← 用它
2. 带 `read:org` 的 PAT，且发布者必须是该组织 Owner

本地手工 `login github` **不得**作为兜底——它会静默发到另一个命名空间。

## 怎么用

```bash
# 1) 先 dry-run：只校验、不写、不发布，打印会发哪些 server
gh workflow run mcp-registry-publish.yml -R FindDataTechnology/mcp-registry-publish

# 2) 确认后正式发布（首次即认领组织命名空间，其后是更新）
gh workflow run mcp-registry-publish.yml -R FindDataTechnology/mcp-registry-publish \
  -f confirm=publish

# 3) 取证
curl -s "https://registry.modelcontextprotocol.io/v0/servers?search=io.github.FindDataTechnology"
```

每周一 03:00 UTC 自动跑一次 dry-run（`schedule`），投影回归会以红 run 暴露，
而不是让 registry 上的列表悄悄过期几个月。

## 发布前门（脚本内强制，均整批不通过则不发布）

| 门 | 判据 |
|---|---|
| 来源是公开投影 | 只消费 `/api/mcp-registry.json`（`groups=[]` 的 server），不读私有源码仓 |
| 零凭据 | 投影侧 `CREDENTIAL_RE` 命中即 `PROJECT_REJECTED`；本仓脚本再验 `headers` 整块缺席 |
| **端点公网可达** | `remotes[].url` 必须 `https://` 且非内网（`mesh.finddata.cn` / 裸 `cheapN` / RFC1918 / loopback / `.local` / `.internal`）——公共目录里放内网地址既是死链，又会永久泄漏内网主机名与端口 |
| 形状合法 | 必填 `name`（反向 DNS，恰好一个 `/`）/`description`/`version` + `remotes[]` |
| last-good | 暂存目录构建 → 整批校验通过才原子替换；失败保留上一份 |

## 与主仓的关系

脚本与 workflow 的**源**在 `FindDataTechnology/fd-craw-private` 的
`.github/workflows/mcp-registry-publish.yml` 与 `scripts/publish-mcp-registry.mjs`。
本仓是它在公开侧的运行副本（因为上面那条额度/可见性约束）。
改逻辑请改主仓那份，再同步过来，不要两边各写一套。
