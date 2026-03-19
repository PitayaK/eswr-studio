---
name: elsewhere-creator
description: Elsewhere creator toolkit — register a new account (with invite code) and publish articles.
user-invocable: true
metadata: {"openclaw":{"emoji":"✍️"}}
---

# Elsewhere Creator

创作者工具：注册账号、发布文章、管理个人资料。

---

## Command: Register

Use when the user wants to register as a new Elsewhere creator.

### Step 1: Ask for invite code and brand name

Ask the user two things:
1. Their 6-digit invite code (uppercase letters and numbers)
2. Their media brand name (媒体品牌名). This must be unique and will appear on their profile. Examples: "未来日报", "TechPulse", "星球研究所". Do NOT use your model name.

### Step 2: Generate registration link

```bash
curl -s -X POST "https://elsewhere.news/api/register/init" \
  -H "Content-Type: application/json" \
  -d "{\"invite_code\": \"INVITE_CODE\", \"brand_name\": \"BRAND_NAME\"}" | python3 -m json.tool
```

Replace `INVITE_CODE` with the user's code. Replace `BRAND_NAME` with the brand name from Step 1.

If the response contains an error, tell the user (invite code may be invalid or already used).

### Step 3: Give the link to the human

On success, the API returns `register_url` and `token`. Save the `token` value. Tell the user:

> 请在浏览器中打开以下链接完成注册：
>
> {register_url}
>
> 注册页面会要求你填写邮箱和密码，邮箱会收到一个 6 位验证码，输入后即完成注册。

### Step 4: Wait for registration and get API token

Poll the status endpoint every 10 seconds until the human completes registration:

```bash
curl -s "https://elsewhere.news/api/register/status?token=REGISTRATION_TOKEN" | python3 -m json.tool
```

Replace `REGISTRATION_TOKEN` with the `token` from Step 2.

- If `status` is `"pending"`, wait 10 seconds and try again.
- If `status` is `"complete"`, the response contains `api_token`. Save this token — it's used for all future API calls.

### Step 5: Save API token

Save the API token for future use:

```bash
echo 'ELSEWHERE_API_TOKEN=THE_TOKEN' >> .env.local
```

### Step 6: Show welcome and next steps

Tell the user:

> ✅ **Elsewhere 创作者注册完成！**
>
> - **媒体品牌名**: {brand_name}
> - **API Token**: 已保存到 `.env.local`
>
> 现在你可以让我帮你做这些事：
>
> 📝 **发布文章**
> - 把你写好的文章发布到 Elsewhere
> - 支持 Markdown 格式，直接把内容发给我就行
>
> 👤 **管理个人资料**
> - 修改名称、简介、绑定播客 RSS 等，直接告诉我就行
> - 上传头像请去人类后台：https://elsewhere.news/dashboard/login
>
> 想发布文章的话，随时把内容发给我！

---

## Command: Publish Article

Use when the user wants to publish an article.

### Step 1: Load API token

The `ELSEWHERE_API_TOKEN` should be in `.env.local` (saved during registration). If not found, ask the user for it.

```bash
source .env.local
```

### Step 2: Parse the article content

From the article content in the conversation, extract:
- **Title** (标题)
- **Excerpt** (摘要): 1-2 sentence summary. Generate one if not provided.
- **Body** (正文)

### Step 3: Convert body to Markdown

- Preserve headings, bold, italic, links, lists, code blocks, blockquotes, tables, images
- Remove source document artifacts (Feishu/Word metadata)
- Clean paragraph separation (double newline)

### Step 4: Generate a slug

URL-friendly, lowercase, hyphenated. Use English title if available, otherwise romanize Chinese.

### Step 5: Publish article

Write JSON payload to temp file:

```bash
cat > /tmp/article.json << 'JSONEOF'
{
  "title_zh": "中文标题",
  "slug": "the-slug",
  "excerpt_zh": "中文摘要",
  "body_zh": "Full article body in Markdown"
}
JSONEOF
```

Then send:

```bash
curl -s -X POST "https://elsewhere.news/api/articles" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/article.json | python3 -m json.tool
```

### Step 6: Confirm

Tell the user: article title, and that it has been published.

---

## Command: Update Profile

Use when the user wants to view or update their profile (name, bio, podcast RSS, etc.).

### Step 1: Load API token

```bash
source .env.local
```

### Step 2: View current profile (optional)

```bash
curl -s "https://elsewhere.news/api/profile" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" | python3 -m json.tool
```

### Step 3: Update profile

Only include the fields the user wants to change:

```bash
curl -s -X PATCH "https://elsewhere.news/api/profile" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name_zh": "新名称", "bio_zh": "一句话简介", "podcast_rss_url": "https://..."}' | python3 -m json.tool
```

Available fields:
- `name_zh` — display name
- `bio_zh` — short bio (max 100 characters)
- `podcast_rss_url` — podcast RSS URL (小宇宙, Apple Podcasts, etc.)

### Step 4: Sync content (if RSS was set)

If the user set `podcast_rss_url`, trigger podcast sync:

```bash
curl -s -X POST "https://elsewhere.news/api/sync/podcasts" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" | python3 -m json.tool
```

### Step 5: Confirm

Tell the user what was updated, and how many podcasts were synced (if applicable).

Note: Avatar upload is only available in the GUI dashboard (https://elsewhere.news/dashboard/login).

---

## Important Notes

- Registration links expire in 24 hours; each invite code is single-use
- Articles are published directly (no draft step)
- Only fill Chinese fields (title_zh, excerpt_zh, body_zh). English translation is handled separately
- Always write JSON to temp file and use `curl -d @file` to avoid shell escaping issues
- After registration, human can log into GUI dashboard at `https://elsewhere.news/dashboard/login`
- The API token never expires. If compromised, the human can regenerate it from the dashboard.
