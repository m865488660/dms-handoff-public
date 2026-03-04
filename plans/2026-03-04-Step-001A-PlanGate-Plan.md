# STEP-001A: PlanGate Workflow (Plan → Review → Execute)

## Context
Current governance (STEP-001) requires HANDOFF.md and PROGRESS_LOG.md updates when code changes. However, AI agents often jump directly to implementation without creating a plan first. This leads to:
- No formal approval gate before code changes
- No plan file requirement in CI
- Difficult to track design decisions

STEP-001A adds a mandatory **plan gate** before any implementation work can be merged.

---

## Goal
Enforce **Plan → Review → Execute** workflow via:
1. CI gate requiring a plan file when code changes
2. PR template requiring plan doc path + checkbox
3. Plan template for future steps

---

## Acceptance Criteria
- [ ] A) CI fails if code changes without a plan file in `docs/todo/*Plan*.md`
- [ ] B) PR template includes plan doc path field + checkbox
- [ ] C) Plan template created at `docs/todo/PLAN_TEMPLATE.md`
- [ ] D) Verification: one failing example + one passing example documented

---

## Files to Create/Modify

### 1. `.github/workflows/require-handoff-progress.yml` (MODIFY)
**Path**: `.github/workflows/require-handoff-progress.yml`

**Change**: Add plan file check after HANDOFF/PROGRESS check

```yaml
# Add after the existing HANDOFF/PROGRESS check:
PLAN_CHANGED=$(echo "$CHANGED" | grep -c '^docs/todo/.*Plan.*\.md$' || true)

if [ "$PLAN_CHANGED" -eq 0 ]; then
  echo "ERROR: Code changed but no plan file in docs/todo/*Plan*.md"
  echo "Create a plan document before implementing code changes."
  exit 1
fi
```

**Full updated workflow**:
```yaml
name: require-handoff-and-progress

on:
  pull_request:

jobs:
  gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Ensure HANDOFF/PROGRESS/PLAN updated when code changes
        shell: bash
        run: |
          set -e
          CHANGED=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)

          echo "Changed files:"
          echo "$CHANGED"

          # What counts as "code changes"
          CODE_CHANGED=$(echo "$CHANGED" | egrep -c '^(api/|worker/|web/|configs/|schemas/|scripts/|tests/)' || true)

          if [ "$CODE_CHANGED" -gt 0 ]; then
            HANDOFF_CHANGED=$(echo "$CHANGED" | grep -c '^docs/HANDOFF\.md$' || true)
            PROGRESS_CHANGED=$(echo "$CHANGED" | grep -c '^docs/PROGRESS_LOG\.md$' || true)
            PLAN_CHANGED=$(echo "$CHANGED" | grep -c '^docs/todo/.*Plan.*\.md$' || true)

            if [ "$HANDOFF_CHANGED" -eq 0 ] || [ "$PROGRESS_CHANGED" -eq 0 ]; then
              echo "ERROR: Code changed but docs/HANDOFF.md or docs/PROGRESS_LOG.md not updated."
              exit 1
            fi

            if [ "$PLAN_CHANGED" -eq 0 ]; then
              echo "ERROR: Code changed but no plan file in docs/todo/*Plan*.md"
              echo "Create a plan document before implementing code changes."
              exit 1
            fi
          fi

          echo "OK"
```

### 2. `.github/pull_request_template.md` (MODIFY)
**Path**: `.github/pull_request_template.md`

**Change**: Add plan doc path field + checkbox at the top

```markdown
## Plan Document（方案文档）
- Plan Doc: `docs/todo/YYYY-MM-DD-Step-XXX-功能-Plan.md`
- [ ] 我已创建方案文档并获得审批（重大变更必须）

---

## Checklist（必选）
- [ ] 我已更新 docs/HANDOFF.md（当前状态快照）
- [ ] 我已追加 docs/PROGRESS_LOG.md（本步完成记录）
- [ ] 我已提供可复现的运行/测试命令
```

### 3. `docs/todo/PLAN_TEMPLATE.md` (CREATE)
**Path**: `docs/todo/PLAN_TEMPLATE.md`

**Content**:
```markdown
# Step-XXX: [功能名称]

## Context（背景）
> 为什么需要这个变更？解决什么问题？

## Goal（目标）
> 可验收的目标，完成后可以勾选

- [ ] A) ...
- [ ] B) ...
- [ ] C) ...

## Design（方案设计）
> 技术方案、架构图、关键代码片段

### 影响文件清单
| 文件 | 操作 | 说明 |
|------|------|------|
| path/to/file | CREATE/MODIFY | 描述 |

### API 变更（如有）
> 新增/修改的 API 端点

### 数据库变更（如有）
> Schema 变更、迁移脚本

## Risk Analysis（风险分析）
> 潜在风险 + 缓解措施

## Test Plan（测试计划）
> 如何验证变更正确性

```bash
# 测试命令
```

## Rollback Plan（回滚方案）
> 如果出问题如何回滚

---

**Approval**: [ ] 已获项目负责人批准

**实施前检查**:
- [ ] 已完整理解现有代码
- [ ] 已回答 KISS 4问题
- [ ] 已获得用户审批
```

### 4. `docs/HANDOFF.md` (MODIFY)
**Path**: `docs/HANDOFF.md`

**Change**: Add PlanGate capability to "当前可演示能力" section

```markdown
- ✅ PlanGate 工作流：CI 强制要求 plan → review → execute
```

### 5. `docs/PROGRESS_LOG.md` (MODIFY)
**Path**: `docs/PROGRESS_LOG.md`

**Change**: Append STEP-001A entry

```markdown
---

## [2026-03-04] STEP-001A：PlanGate 工作流落地（Plan → Review → Execute）
**Owner**：Antigravity
**Goal（可验收）**：
- A) CI 失败：代码变更但没有 plan 文件
- B) PR 模板包含 plan 文档路径 + checkbox
- C) Plan 模板创建

**Done**：
- ✅ 修改 `.github/workflows/require-handoff-progress.yml`：增加 plan 文件检查
- ✅ 修改 `.github/pull_request_template.md`：增加 plan 路径 + checkbox
- ✅ 创建 `docs/todo/PLAN_TEMPLATE.md`

**Files Changed**：
- .github/workflows/require-handoff-progress.yml
- .github/pull_request_template.md
- docs/todo/PLAN_TEMPLATE.md
- docs/HANDOFF.md
- docs/PROGRESS_LOG.md

**Commands / Tests**：
- （见下方 Verification）

**Result（可演示点）**：
- 代码变更必须伴随 plan 文件
- CI 自动检查 plan 文件存在

**Known Issues / Risks**：
- 小型修复（typo、文档）也需要 plan 文件，可能过于严格

**Next**：
- Step-003：SMB 扫描支持
```

---

## Verification

### Failing Example (CI should FAIL)
1. Create branch `test/no-plan-code-change`
2. Modify `api/main.py` (add a comment)
3. Update `docs/HANDOFF.md` and `docs/PROGRESS_LOG.md`
4. **Do NOT** create any plan file
5. Open PR
6. **Expected**: CI fails with "no plan file in docs/todo/*Plan*.md"

### Passing Example (CI should PASS)
1. Create branch `test/with-plan-code-change`
2. Create `docs/todo/2026-03-04-Test-Plan.md` with minimal content
3. Modify `api/main.py`
4. Update `docs/HANDOFF.md` and `docs/PROGRESS_LOG.md`
5. Open PR
6. **Expected**: CI passes all checks

---

## Implementation Order
1. Create `docs/todo/PLAN_TEMPLATE.md`
2. Modify `.github/workflows/require-handoff-progress.yml`
3. Modify `.github/pull_request_template.md`
4. Update `docs/HANDOFF.md`
5. Update `docs/PROGRESS_LOG.md`

---

## Summary

| File | Action | Purpose |
|------|--------|---------|
| `.github/workflows/require-handoff-progress.yml` | MODIFY | Add plan file check |
| `.github/pull_request_template.md` | MODIFY | Add plan path + checkbox |
| `docs/todo/PLAN_TEMPLATE.md` | CREATE | Template for future plans |
| `docs/HANDOFF.md` | MODIFY | Add PlanGate capability |
| `docs/PROGRESS_LOG.md` | MODIFY | Append STEP-001A entry |
