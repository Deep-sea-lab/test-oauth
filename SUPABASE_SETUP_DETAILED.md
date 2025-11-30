# OAuth-Between Supabase 配置指南

## 概述
本项目现已使用 Supabase 作为临时 OAuth token 存储方案，替代了 Netlify 无状态实例的内存存储。这样可以保证 token 在多个 Netlify 函数实例之间可靠地被读取。

## 初始化检查清单

- [ ] 创建 Supabase 项目
- [ ] 获取 SUPABASE_URL 和 SUPABASE_ANON_KEY
- [ ] 在 Supabase 中创建 `oauth_tokens` 表
- [ ] 设置行级安全（RLS）策略
- [ ] 在 Netlify 中配置环境变量
- [ ] 测试 OAuth 流程

## 1. 创建 Supabase 项目

1. 访问 [https://app.supabase.com](https://app.supabase.com) 并登录（如无账号先注册）
2. 点击 "New Project" 创建新项目
3. 填写项目信息：
   - **Project name**: 例如 `02engine-oauth`
   - **Database password**: 设置强密码（保存此密码！）
   - **Region**: 选择离你最近的地区（例如新加坡、东京）
4. 等待项目创建完成（通常需要 1-2 分钟）
5. 项目创建后，在左侧导航栏找到项目 URL 和 API Keys：
   - 点击 "Settings" → "API"
   - 复制 **URL**（SUPABASE_URL）
   - 复制 **anon** 密钥下的值（SUPABASE_ANON_KEY）

## 2. 创建 OAuth Token 表

### 方法 A：使用 Supabase UI（推荐）

1. 在 Supabase 仪表板中，点击左侧 "SQL Editor"
2. 点击 "New Query"
3. 将以下 SQL 代码复制到编辑器中：

```sql
-- 创建 oauth_tokens 表
CREATE TABLE oauth_tokens (
  id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  hash VARCHAR(255) UNIQUE NOT NULL,
  token TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- 创建索引以加快查询
CREATE INDEX idx_oauth_tokens_hash ON oauth_tokens(hash);
CREATE INDEX idx_oauth_tokens_expires_at ON oauth_tokens(expires_at);

-- 设置行级安全（RLS）
ALTER TABLE oauth_tokens ENABLE ROW LEVEL SECURITY;

-- 允许匿名用户读取未过期的 token
CREATE POLICY "Allow anon to read valid tokens"
  ON oauth_tokens
  FOR SELECT
  TO anon
  USING (expires_at > NOW());

-- 允许匿名用户插入新 token
CREATE POLICY "Allow anon to insert tokens"
  ON oauth_tokens
  FOR INSERT
  TO anon
  WITH CHECK (true);

-- 允许匿名用户更新 token
CREATE POLICY "Allow anon to update tokens"
  ON oauth_tokens
  FOR UPDATE
  TO anon
  USING (true);

-- 允许匿名用户删除过期 token
CREATE POLICY "Allow anon to delete expired tokens"
  ON oauth_tokens
  FOR DELETE
  TO anon
  USING (expires_at <= NOW());
```

4. 点击 "Run" 或按 `Ctrl+Enter` 执行 SQL
5. 应该看到成功提示（✓ Query successful）

### 方法 B：验证表创建成功

1. 点击左侧 "Table Editor"
2. 刷新或重新加载页面
3. 应该能看到 `oauth_tokens` 表出现在表列表中

## 3. 配置 Netlify 环境变量

### 步骤：

1. 登录 [Netlify 仪表板](https://app.netlify.com)
2. 进入 `idyllic-kangaroo-a50663.netlify.app` 项目（或你部署的项目）
3. 点击 "Site settings" → "Build & deploy" → "Environment"
4. 点击 "Edit variables"
5. 添加两个环境变量：
   - **Name**: `SUPABASE_URL`  
     **Value**: `https://your-project.supabase.co`（从第 1 步复制）
   - **Name**: `SUPABASE_ANON_KEY`  
     **Value**: `你的匿名密钥`（从第 1 步复制）
6. 点击 "Save"

## 4. 验证部署

1. 返回 Netlify 仪表板
2. 点击 "Deployments"
3. 应该能看到最新的部署状态
4. 等待构建完成（显示绿色对号）

## 5. 测试 OAuth 流程

### 测试步骤：

1. 在桌面端启动应用
2. 触发 OAuth 认证（例如登录 GitHub）
3. 验证以下日志输出：
   - 前端：`[OAuth-Between] Token sent to cloud (Supabase) successfully`
   - 桌面端：`[Desktop OAuth] Successfully got token`

### 检查数据库：

1. 在 Supabase 仪表板中点击 "Table Editor"
2. 选择 `oauth_tokens` 表
3. 应该能看到刚创建的记录（包含 hash、token、expires_at）

## 6. 工作流程

1. **前端（oauth-between/index.html）**：
   - 用户授权后获得 token
   - 调用 `/store-token` POST 端点（将 token 存储到 Supabase，10 分钟过期）
   - 日志：`[OAuth-Between] Token sent to cloud (Supabase) successfully`

2. **桌面应用（src-main/windows/editor.js）**：
   - 轮询 `/temp-token/{hash}` GET 端点
   - 从 Supabase 查询并返回 token
   - 日志：`[Desktop OAuth] Successfully got token`

3. **过期清理**：
   - Token 10 分钟后自动过期
   - Supabase 会阻止读取过期记录（由 RLS 策略控制）

## 7. 故障排查

### 问题 1："Token未找到或已过期" 错误

**症状**：
- 前端成功存储 token，但桌面端轮询时返回 404

**检查清单**：
- [ ] `SUPABASE_URL` 和 `SUPABASE_ANON_KEY` 已正确设置到 Netlify
- [ ] 在 Supabase 仪表板中能看到新记录
- [ ] 表 `oauth_tokens` 的 RLS 策略已启用
- [ ] 检查 Netlify 函数日志（Site settings → Functions）

**解决方案**：
```bash
# 1. 重新部署（强制清除缓存）
git commit --allow-empty -m "Force rebuild"
git push origin main

# 2. 在 Supabase 中手动测试表访问权限
# 点击 "SQL Editor" → "New Query" 执行：
SELECT * FROM oauth_tokens LIMIT 10;
```

### 问题 2：函数返回 500 错误

**症状**：
- 前端调用 `/store-token` 返回 500 Internal Server Error

**检查步骤**：
1. 查看 Netlify Functions 日志：
   - Netlify 仪表板 → Site settings → Functions → View logs
2. 检查错误信息（通常是连接或权限问题）
3. 验证环境变量是否已正确保存

**常见原因**：
- Supabase URL 或密钥不正确
- 网络连接问题
- Supabase 项目异常

### 问题 3：行级安全（RLS）拒绝访问

**症状**：
- Netlify 函数日志显示 `permission denied`

**解决方案**：
重新执行 RLS 策略 SQL：
```sql
-- 在 Supabase SQL Editor 中执行
ALTER TABLE oauth_tokens ENABLE ROW LEVEL SECURITY;

-- 删除旧策略（如果存在）
DROP POLICY IF EXISTS "Allow anon to read valid tokens" ON oauth_tokens;
DROP POLICY IF EXISTS "Allow anon to insert tokens" ON oauth_tokens;
DROP POLICY IF EXISTS "Allow anon to update tokens" ON oauth_tokens;
DROP POLICY IF EXISTS "Allow anon to delete expired tokens" ON oauth_tokens;

-- 重新创建策略
CREATE POLICY "Allow anon to read valid tokens"
  ON oauth_tokens FOR SELECT TO anon
  USING (expires_at > NOW());

CREATE POLICY "Allow anon to insert tokens"
  ON oauth_tokens FOR INSERT TO anon
  WITH CHECK (true);

CREATE POLICY "Allow anon to update tokens"
  ON oauth_tokens FOR UPDATE TO anon
  USING (true);

CREATE POLICY "Allow anon to delete expired tokens"
  ON oauth_tokens FOR DELETE TO anon
  USING (expires_at <= NOW());
```

## 8. 性能优化

- Token 有效期为 **10 分钟**（可在 `store-token.js` 中调整）
- 过期 token 会自动被数据库过期删除（Supabase 自动清理）
- 索引 `idx_oauth_tokens_hash` 确保 O(1) 查询性能

## 9. 安全注意事项

- ✅ 仅使用 **ANON_KEY**（权限受限）
- ✅ Token 自动 10 分钟后过期
- ✅ RLS 策略确保匿名用户只能读取未过期的 token
- ⚠️ 建议：定期检查 `oauth_tokens` 表的大小，清理过期数据
- ⚠️ 生产环境：考虑添加 IP 白名单或速率限制

## 10. 后续维护

### 监控表大小：
```sql
SELECT COUNT(*) as total_records,
       COUNT(CASE WHEN expires_at > NOW() THEN 1 END) as valid_records,
       COUNT(CASE WHEN expires_at <= NOW() THEN 1 END) as expired_records
FROM oauth_tokens;
```

### 手动清理过期数据：
```sql
DELETE FROM oauth_tokens WHERE expires_at <= NOW();
```

## 11. 回滚（如需恢复旧方案）

若需改回内存存储：
1. 编辑 `oauth-between/netlify/functions/store-token.js` 和 `temp-token.js`
2. 参考 Git 历史恢复旧实现
3. 删除 Supabase 相关代码和环境变量
4. 推送更改到 Netlify 重新部署
