# Progress Log（追加式，不改历史）

## [2026-03-04] STEP-001：协作治理落地（CLAUDE协作协议 + 交接文档 + CI门禁）
**Owner**：Antigravity
**Goal（可验收）**：改代码必须更新 HANDOFF/PROGRESS_LOG，否则CI失败
**Done**：
- ✅ 新增 docs/HANDOFF.md 与 docs/PROGRESS_LOG.md
- ✅ 更新 CLAUDE.md：加入 AI 协作协议（强制）
- ✅ 新增 CONTRIBUTING.md
- ✅ 新增 PR 模板 + CI 门禁
**Files Changed**：
- docs/HANDOFF.md
- docs/PROGRESS_LOG.md
- CLAUDE.md
- CONTRIBUTING.md
- .github/pull_request_template.md
- .github/workflows/require-handoff-progress.yml
**Commands / Tests**：
- （如：CI自动跑过）
**Result（可演示点）**：
- 开PR时模板要求勾选
- 改代码但未更新文档时 CI 失败
**Known Issues / Risks**：
- （无）
**Next**：
- Step-002：项目骨架一键启动（M0基建）

---

## [2026-03-04] STEP-001B：公共仓库发布脚本（脱敏发布 HANDOFF/PROGRESS_LOG）
**Owner**：Antigravity
**Goal（可验收）**：
- A) PowerShell脚本 `scripts/publish_handoff.ps1` 支持脱敏发布
- B) 发布 HANDOFF.md、PROGRESS_LOG.md、最新Plan文件到公共仓库
- C) 支持 DryRun 模式预览

**Done**：
- ✅ 创建 `scripts/publish_handoff.ps1`
  - 参数：-PublicRepoDir, -StepId, -Message, -DryRun
  - 复制：HANDOFF.md, PROGRESS_LOG.md, 最新 *Plan.md
  - Git 操作：add → commit → push
- ✅ 更新 `docs/CLAUDE.md`：添加 Public Handoff Publishing 协议
- ✅ 更新 `CONTRIBUTING.md`：添加发布流程和脱敏要求

## [2026-03-04] HOTFIX：publish_handoff.ps1 兼容 git stderr / no-op commit + 忽略 .claude/

**Owner**：Antigravity / Claude Code
**Goal**：publish 脚本不因 git 输出误报失败；忽略 .claude/ 防止污染工作区
**Done**：
- ✅ publish_handoff.ps1：允许 “nothing to commit”，并捕获 push 输出
- ✅ .gitignore：忽略 .claude/
**Files Changed**：
- scripts/publish_handoff.ps1
- .gitignore
**Commands / Tests**：
- .\scripts\publish_handoff.ps1 -PublicRepoDir C:\dms-handoff-public -StepId "Step-TEST" -Message "plan"
**Result**：
- 无变更时显示 “No changes to commit.” 仍能正常结束
**Next**：
- Step-001A：PlanGate（Plan → Review → Execute）门禁落地

## [2026-03-04] HOTFIX: publish_handoff.ps1 兼容 PowerShell 7 stderr 误报

**Owner**：Claude Code
**Done**：
- ✅ 禁用 PS7 的 native stderr → error record 映射，避免 git push 误报失败
**Files Changed**：
- scripts/publish_handoff.ps1
**Next**：
- Step-001A：PlanGate 落地

## [2026-03-04] HOTFIX: publish_handoff.ps1 使用 Start-Process 执行 git push（避免 PS7 NativeCommandError）

**Owner**：Claude Code
**Done**：
- ✅ git push 改为 Start-Process + ExitCode 判定，避免 PS7 将 stderr 当异常
**Files Changed**：
- scripts/publish_handoff.ps1
**Next**：
- Step-001A：PlanGate 落地

## [2026-03-04] HOTFIX: 修复 publish_handoff.ps1 try/finally 语法错误

**Owner**：Claude Code
**Done**：修复 try/finally 花括号配对，DryRun 可正常执行
**Files Changed**：scripts/publish_handoff.ps1


**Files Changed**：
- scripts/publish_handoff.ps1（新增）
- docs/CLAUDE.md
- CONTRIBUTING.md
- docs/HANDOFF.md
- docs/PROGRESS_LOG.md

**Commands / Tests**：
```powershell
# DryRun 模式测试
.\scripts\publish_handoff.ps1 -DryRun

# 实际发布（Plan阶段后）
.\scripts\publish_handoff.ps1 -PublicRepoDir C:\dms-handoff-public -StepId "Step-XXX" -Message "plan"

# 实际发布（Execute阶段后）
.\scripts\publish_handoff.ps1 -PublicRepoDir C:\dms-handoff-public -StepId "Step-XXX" -Message "done"
```

**Result（可演示点）**：
- DryRun 模式显示将要复制的文件，不执行任何更改
- 脱敏要求明确：禁止发布 SMB路径、LAN IP、设备SN、凭证等
- 公共仓库：https://github.com/m865488660/dms-handoff-public

**Known Issues / Risks**：
- 仅支持 Windows PowerShell
- 需要手动确保公共仓库已 clone 到本地

**Next**：
- Step-002：项目骨架一键启动（M0基建）

---

## [2026-03-04] STEP-001A：PlanGate 工作流落地（Plan → Review → Execute）
**Owner**：Antigravity
**Goal（可��收）**：
- A) CI 失败：代码变更但没有 plan 文件
- B) PR 模板包含 plan 文档路径 + checkbox
- C) Plan 模板创建

**Done**：
- ✅ 修改 `.github/workflows/require-handoff-progress.yml`：增加 plan 文件检查
  - 正则：`^docs/todo/.*(Step|step)-.*Plan.*\.md$`
- ✅ 修改 `.github/pull_request_template.md`：增加 plan 路径 + checkbox
- ✅ 创建 `docs/todo/PLAN_TEMPLATE.md`：方案文档模板
- ✅ 更新 `docs/CLAUDE.md`：添加 Plan → Review → Execute 协议
- ✅ 更新 `docs/HANDOFF.md`：添加 PlanGate 能力说明

**Files Changed**：
- .github/workflows/require-handoff-progress.yml
- .github/pull_request_template.md
- docs/todo/PLAN_TEMPLATE.md
- docs/todo/2026-03-04-Step-001A-PlanGate-Plan.md
- docs/CLAUDE.md
- docs/HANDOFF.md
- docs/PROGRESS_LOG.md

**Commands / Tests**：
```bash
# 验证 CI 正则
echo "docs/todo/2026-03-04-Step-001A-PlanGate-Plan.md" | grep -cE '^docs/todo/.*(Step|step)-.*Plan.*\.md$'
# → 1 (匹配)
```

**Result（可演示点）**：
- 代码变更必须伴随 `docs/todo/*Step*-*.Plan*.md` 文件
- CI 自动检查 plan 文件存在
- PR 模板要求填写 plan 文档路径

**Known Issues / Risks**：
- 小型修复（typo、文档）也需要 plan 文件，可能过于严格
- 可考虑后续增加 `docs-only` 标签跳过 plan 检查

**Next**：
- Step-003：SMB 扫描支持

---

## [2026-03-05] STEP-002：DMS 可运行骨架（Docker Compose 全栈）
**Owner**：Antigravity
**Goal（可验收）**：
- A) `docker compose up -d` 一键启动 5 服务（postgres, redis, api, worker, web）
- B) API 健康检查 `GET /health` 返回 200
- C) 可创建端点、提交扫描任务、查看检测到的数据集
- D) 数据集分类正确（raw / result / hybrid / gcp）
- E) CI 门禁通过

**Done**：
- ✅ Docker Compose 配置（5 服务，健康检查，卷挂载）
- ✅ PostgreSQL 数据库初始化（pgcrypto, 4 表 + 索引）
- ✅ FastAPI 后端（/health, /endpoints, /tasks/scan, /datasets）
- ✅ RQ Worker 扫描逻辑（签名规则分类，只读文件访问）
- ✅ Next.js Web UI（3 页面：端点、任务、数据集）
- ✅ 示例数据（sample_data/ 包含 4 种测试数据集）
- ✅ 端到端验证：4 数据集正确检测

**Files Changed**：
- docker-compose.yml（新增）
- .env.example（新增）
- .dockerignore（新增）
- api/Dockerfile, main.py, database.py, models.py, init_db.sql（新增）
- api/routes/endpoints.py, tasks.py, datasets.py（新增）
- worker/Dockerfile, worker.py, scan.py, requirements.txt（新增）
- web/Dockerfile, package.json, next.config.js（新增）
- web/app/layout.js, page.js, globals.css（新增）
- web/app/endpoints/page.js, tasks/page.js, datasets/page.js（新增）
- web/lib/api.js（新增）
- sample_data/（4 个测试数据集目录）
- configs/settings.yaml（新增）
- README.md（更新）
- docs/HANDOFF.md, docs/PROGRESS_LOG.md

**Commands / Tests**：
```bash
# 启动全栈
docker compose up -d --build
docker compose ps  # 5 服务 running

# API 健康检查
curl http://localhost:8090/health
# → {"status":"ok","version":"0.1.0"}

# 创建端点 + 扫描
curl -X POST http://localhost:8090/endpoints \
  -H "Content-Type: application/json" \
  -d '{"name":"Local Test","type":"local","config":{"roots":["/data"]}}'

curl -X POST http://localhost:8090/tasks/scan \
  -H "Content-Type: application/json" \
  -d '{"endpoint_id":"<ENDPOINT_ID>"}'

# 等待完成，验证数据集
curl http://localhost:8090/datasets
# → 4 datasets: raw, result, hybrid, raw(gcp)
```

**Result（可演示点）**：
- 一键启动：`docker compose up -d` 即可运行全栈
- Web UI：http://localhost:3000 可视化管理
- 扫描结果：4 种数据集类型正确识别
- 只读扫描：无任何文件修改

**As-built Notes**：
- 数据库初始化使用 `/docker-entrypoint-initdb.d/` 挂载（非 Alembic）
- UUID 使用 `gen_random_uuid()` 需要 pgcrypto 扩展
- Web Dockerfile 需 `output: 'standalone'` 用于生产镜像
- Worker 使用同步 psycopg2（非 asyncpg）

**Known Issues / Risks**：
- SMB 端点类型返回 501（待 Step-003）
- 无认证/授权
- 无分页（数据量大时性能问题）
- 无 WebSocket 实时更新

**Next**：
- Step-003：SMB/CIFS 网络扫描支持

---

## [2026-03-06] STEP-003 PlanGate：SMB/CIFS 网络扫描支持 — PLAN 阶段
**Owner**：Antigravity
**Status**：PLAN / REVIEW
**Goal（可验收）**：
- A) SMB端点类型激活（API不再返回501）
- B) Worker使用Python SMB库直接扫描，无需OS挂载
- C) 凭证安全存储（DB JSONB），不泄露到日志/公开文档
- D) Dockerized Samba集成测试

**Plan Document**：`docs/todo/2026-03-06-1500-Step-003-SMB-Scan-Plan.md`

**Key Design Decisions**：
- SMB库：`smbprotocol`（SMB2/SMB3，纯Python）
- 凭证存储：DB JSONB + 环境变量fallback
- 集成测试：Dockerized Samba容器复用sample_data/
- 分类规则：保持raw/result/hybrid + has_gcp

**Files Changed (Plan only)**：
- docs/todo/2026-03-06-1500-Step-003-SMB-Scan-Plan.md（新增）
- docs/HANDOFF.md（更新当前阶段）
- docs/PROGRESS_LOG.md（本条目）

**Pending Execution Tasks**：
- [ ] Phase 1: Database & Models（credentials列，SMB配置模型）
- [ ] Phase 2: API Routes（移除501，credentials端点）
- [ ] Phase 3: Worker Walker Abstraction（LocalWalker/SMBWalker）
- [ ] Phase 4: Integration Testing（docker-compose.test.yml）
- [ ] Phase 5: Documentation & Publish

**Next**：
- Step-003 Execute 阶段（待审批后执行）