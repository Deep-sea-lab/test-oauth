# OAuth-Between Supabase 配置指南

## 概述
本项目现已使用 Supabase 作为临时 OAuth token 存储方案，替代了 Netlify 无状态实例的内存存储。这样可以保证 token 在多个 Netlify 函数实例之间可靠地被读取。

## 1. 创建 Supabase 项目

1. 访问 [https://app.supabase.com](https://app.supabase.com) 并登录
2. 创建新项目或使用现有项目
3. 获取以下信息：
   - **SUPABASE_URL**: 项目 URL（格式：`https://your-project.supabase.co`）
   - **SUPABASE_ANON_KEY**: 匿名访问密钥

## 2. 创建 OAuth Token 表

在 Supabase 数据库中执行以下 SQL 创建表：

```sql
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

-- 设置行级安全（RLS）策略
ALTER TABLE oauth_tokens ENABLE ROW LEVEL SECURITY;

-- 允许匿名用户读写自己的 token（基于时间戳）
CREATE POLICY "Allow anon to read valid tokens"
  ON oauth_tokens
  FOR SELECT
  TO anon
  USING (expires_at > NOW());

CREATE POLICY "Allow anon to insert tokens"
  ON oauth_tokens
  FOR INSERT
  TO anon
  WITH CHECK (true);

CREATE POLICY "Allow anon to update tokens"
  ON oauth_tokens
  FOR UPDATE
  TO anon
  USING (true);

CREATE POLICY "Allow anon to delete expired tokens"
  ON oauth_tokens
  FOR DELETE
  TO anon
  USING (expires_at <= NOW());
```

## 3. 配置 Netlify 环境变量

在 Netlify 项目设置中（Site settings → Build & deploy → Environment）添加：

```
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_ANON_KEY=your-anon-key-here
```

## 4. 部署函数依赖

确保 `oauth-between/netlify/functions/` 中的 Node.js 函数有 Supabase 依赖：

```bash
cd oauth-between/netlify/functions
npm install @supabase/supabase-js
```

## 5. 工作流程

1. **前端（oauth-between/index.html）**：
   - 用户授权后获得 token
   - 调用 `/store-token` POST 端点（将 token 存储到 Supabase，10 分钟过期）

2. **桌面应用（src-main/windows/editor.js）**：
   - 轮询 `/temp-token/{hash}` GET 端点
   - 从 Supabase 获取已验证的 token

3. **过期清理**：
   - Token 10 分钟后自动过期
   - 桌面端可手动删除过期记录（见 `temp-token.js`）

## 6. 故障排查

### "Token未找到或已过期" 错误
- 检查 Supabase URL 和密钥是否正确
- 确认 `oauth_tokens` 表存在
- 确保行级安全（RLS）策略允许匿名访问

### 函数无法连接 Supabase
- 检查网络连接
- 验证环境变量是否设置
- 查看 Netlify 函数日志

### Token 过期太快
- 检查系统时钟是否同步
- 在 `store-token.js` 中调整过期时间（当前为 10 分钟）

## 7. 安全注意事项

- 仅使用 **ANON_KEY**（限制权限）
- 定期检查 `oauth_tokens` 表的大小
- 考虑添加 IP 白名单或速率限制
- 不在代码中硬编码密钥

## 8. 切换回其他存储方案

如需改回内存存储或使用其他数据库：
- 修改 `store-token.js` 和 `temp-token.js`
- 参考 Git 历史中的之前实现
