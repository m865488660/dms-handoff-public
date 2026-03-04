# Handoff（当前状态）

## 当前可演示能力
- ✅ ���键启动全栈：`docker compose up -d` (postgres, redis, api, worker, web)
- ✅ API健康检查：`GET /health` 返回 `{"status":"ok","version":"0.1.0"}`
- ✅ 端点管理：创建/列出本地存储端点
- ✅ 扫描任务：提交扫描 → Worker检测数据集 → 结果存入DB
- ✅ 数据集分类：raw / result / hybrid / has_gcp 四种类型识别
- ✅ Web UI：端点管理、任务大厅、数据集列表（含筛选）
- ✅ 示例数据：sample_data/ 包含4种测试数据集
- ✅ 公共仓库发布：scripts/publish_handoff.ps1 脱敏发布文档

## 当前阻塞/风险
- 无重大阻塞
- SMB端点类型尚未实现（计划Step-003）
- 尚未实现用��认证（计划后续迭代）

## 下一步（按优先级）
1. Step-003：SMB扫描支持（远程网络共享）
2. Step-004：元数据解析（metadata.yaml、device_info.json解析入库）
3. Step-005：数据集去重（基于fingerprint跨端点合并）

## 最新启动方式（会随迭代更新）
```bash
# 启动全部服务
docker compose up -d --build

# 验证健康状态
curl http://localhost:8090/health

# 访问Web UI
open http://localhost:3000

# 停止服务
docker compose down
```

### 端口映射
| 服务 | 端口 | 说明 |
|------|------|------|
| API | 8090 | FastAPI后端 |
| Web | 3000 | Next.js前端 |
| Postgres | 5432 | 数据库 |
| Redis | 6379 | 消息队列 |

### 公共仓库发布
```powershell
# 发布到公共仓库
.\scripts\publish_handoff.ps1 -PublicRepoDir C:\dms-handoff-public -StepId "Step-XXX" -Message "done"
```
