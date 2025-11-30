# Supabase 快速初始化指南

完整指南请参考 `SUPABASE_SETUP_DETAILED.md`。本文件是快速参考。

## 30 秒快速开始

### 1️⃣ Supabase 中创建表（复制粘贴即可）

访问 Supabase SQL Editor，执行此代码：

```sql
CREATE TABLE oauth_tokens (
  id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  hash VARCHAR(255) UNIQUE NOT NULL,
  token TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  expires_at TIMESTAMP WITH TIME ZONE NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_oauth_tokens_hash ON oauth_tokens(hash);
CREATE INDEX idx_oauth_tokens_expires_at ON oauth_tokens(expires_at);

ALTER TABLE oauth_tokens ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Allow anon to read valid tokens"
  ON oauth_tokens FOR SELECT TO anon USING (expires_at > NOW());
CREATE POLICY "Allow anon to insert tokens"
  ON oauth_tokens FOR INSERT TO anon WITH CHECK (true);
CREATE POLICY "Allow anon to update tokens"
  ON oauth_tokens FOR UPDATE TO anon USING (true);
CREATE POLICY "Allow anon to delete expired tokens"
  ON oauth_tokens FOR DELETE TO anon USING (expires_at <= NOW());
```

✅ 看到绿色对号 = 成功

### 2️⃣ 获取凭据

在 Supabase 中：Settings → API
- 复制 **URL** → 设为 `SUPABASE_URL`
- 复制 **anon** 密钥 → 设为 `SUPABASE_ANON_KEY`

### 3️⃣ Netlify 中配置环境变量

Site settings → Build & deploy → Environment → Edit variables
- `SUPABASE_URL=https://...`
- `SUPABASE_ANON_KEY=...`

保存后，Netlify 会自动重新部署。

### 4️⃣ 验证成功

桌面端触发 OAuth，应该看到：
- 前端日志：`Token sent to cloud (Supabase) successfully`
- Supabase Table Editor 中能看到新记录

## 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|--------|
| `permission denied` | RLS 策略未设置 | 重新执行上面的 SQL |
| `500 Internal Server Error` | 环境变量不对 | 检查 Netlify 环境变量 |
| `Token未找到或已过期` | 表不存在或数据丢失 | 检查 Supabase Table Editor |

## 文件清单

- ✅ `oauth-between/netlify/functions/store-token.js` - 已配置 Supabase
- ✅ `oauth-between/netlify/functions/temp-token.js` - 已配置 Supabase
- ✅ `oauth-between/netlify.toml` - 已配置依赖安装插件
- ✅ `oauth-between/netlify/functions/package.json` - 已添加 Supabase 依赖

## 需要帮助？

- 详细步骤：参考 `SUPABASE_SETUP_DETAILED.md`
- 故障排查：见上文"常见错误"或 DETAILED 指南的第 7 节
