---
name: elsewhere-creator
description: Elsewhere creator toolkit — register a new account (with invite code) and publish articles.
user-invocable: true
metadata: {"openclaw":{"emoji":"✍️"}}
---

# Elsewhere Creator

创作者工具：注册账号、发布文章、管理个人资料。

> **每次使用本 Skill 前，请先执行以下命令检查是否有更新：**
> ```bash
> curl -s https://raw.githubusercontent.com/PitayaK/elsewhere-creator/main/SKILL.md | head -5
> ```
> 如果输出的版本号与你本地缓存的不同，请重新读取完整文件：
> ```bash
> curl -s https://raw.githubusercontent.com/PitayaK/elsewhere-creator/main/SKILL.md
> ```

**当前版本：v1.7**

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

### Step 4: Upload images

If the article contains images (either as external URLs or local files), upload each one first to get permanent URLs.

**Upload from URL** (for images from WeChat, Feishu, etc.):

```bash
curl -s -X POST "https://elsewhere.news/api/upload" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com/image.jpg"}' | python3 -m json.tool
```

**Upload local file**:

```bash
curl -s -X POST "https://elsewhere.news/api/upload" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
  -F "file=@/path/to/image.jpg" | python3 -m json.tool
```

Both return `{"url": "https://...public-url...", "path": "..."}`.

Replace all image URLs in the Markdown body with the returned `url` values. Use the first uploaded image as `cover_image_url` if no cover is specified.

### Step 5: Generate a slug

URL-friendly, lowercase, hyphenated. Use English title if available, otherwise romanize Chinese.

### Step 6: Publish article

Write JSON payload to temp file:

Translate the title and excerpt to English yourself before writing the JSON. Do NOT translate the body — it will be translated automatically after publishing.

```bash
cat > /tmp/article.json << 'JSONEOF'
{
  "title_zh": "中文标题",
  "title_en": "English Title",
  "slug": "the-slug",
  "excerpt_zh": "中文摘要",
  "excerpt_en": "English excerpt",
  "body_zh": "Full article body in Markdown",
  "cover_image_url": "https://...uploaded-cover-url..."
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

### Step 7: Confirm

Tell the user: article title, and that it has been published.

---

## Command: Import from WeChat

Use when the user shares a WeChat article URL (mp.weixin.qq.com) and wants to publish it to Elsewhere.

### Step 1: Load API token

```bash
source .env.local
```

### Step 2: Import the article

```bash
curl -s -X POST "https://elsewhere.news/api/import" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "WECHAT_ARTICLE_URL"}' | python3 -m json.tool
```

The API returns: `title`, `content` (Markdown with images already uploaded), `cover_image_url`, `published_at` (original publish date from WeChat), and image counts.

### Step 3: Clean up and reformat the Markdown

The import API gives you a raw Markdown conversion of the original article. Before publishing, clean it up into proper Markdown. The goal is a clean, consistent layout — **do not change any text content**.

**What you SHOULD do:**
- Identify bold paragraphs that are clearly section headers (person names, chapter titles, topic labels) and upgrade from `**bold**` to `## heading`
  - SHOULD become `##`: `**曹曦 @MONOLITH**`, `**第一章**`, `**结语**`
  - STAY bold: `**以下排名不分先后，按姓名首字母**`, `**注：本文仅供参考**`
- Remove WeChat-specific embed placeholders that can't display on Elsewhere:
  - Mini program cards (usually lines like `[小程序]` or `[视频号]` or similar text artifacts)
  - Video embeds (lines that are just a video URL with no meaningful text)
  - QR code images are OK to keep — just leave them as regular `![]()` images
- Remove excessive blank lines (max 1 blank line between paragraphs)
- Clean up stray punctuation or formatting artifacts from the HTML conversion

**What you MUST NOT do:**
- Change, rewrite, summarize, or remove any text content
- Change the order of paragraphs or images
- Add any new content that wasn't in the original

Show the user the extracted title and a brief preview. Ask if they want to publish directly or make changes first.

If ready to publish:

**Step 3a: Translate title and excerpt yourself**

Translate the title and excerpt to English yourself (do not call any API). Do NOT translate the body — it will be translated automatically after publishing.

**Step 3b: Save and publish**

```bash
# Save import result
curl -s -X POST "https://elsewhere.news/api/import" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "WECHAT_ARTICLE_URL"}' > /tmp/import_result.json

# Build article JSON using the EXACT content returned by import API
python3 -c "
import json, re, sys
r = json.load(open('/tmp/import_result.json'))
title = r['title']
content = r['content']  # Use this EXACTLY — images are already uploaded
cover = r.get('cover_image_url', '')
pub = r.get('published_at', '')
slug = re.sub(r'[^a-z0-9]+', '-', title.lower())[:50].strip('-') or f'article-{int(__import__(\"time\").time())}'
excerpt = r.get('excerpt', '')  # First meaningful paragraph from import API
article = {'title_zh': title, 'slug': slug, 'excerpt_zh': excerpt, 'body_zh': content, 'cover_image_url': cover, 'published_at': pub}
json.dump(article, open('/tmp/article.json', 'w'), ensure_ascii=False)
print('slug:', slug)
"
```

Then open `/tmp/article.json`, add your translations:
- `title_en`: your English translation of the title
- `excerpt_en`: your English translation of the excerpt
- `body_en`: your English translation of the body (omit if too long)

Then publish:

```bash
curl -s -X POST "https://elsewhere.news/api/articles" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/article.json | python3 -m json.tool
```

**CRITICAL**: Always use the `content` field from the import API directly as `body_zh`. Do NOT re-fetch, re-parse, or rewrite the content — the images are already uploaded and embedded as Markdown image links in `content`. Rewriting it will break all images.

**If the response contains `failed_images`** (images blocked by anti-hotlink server-side), download and re-upload them locally:

```bash
# For each URL in failed_images, download locally and upload to Elsewhere
for img_url in <failed_image_urls>; do
  curl -s -o /tmp/img_temp.jpg "$img_url" \
    -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36" \
    -H "Referer: https://mp.weixin.qq.com/"

  new_url=$(curl -s -X POST "https://elsewhere.news/api/upload" \
    -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
    -F "file=@/tmp/img_temp.jpg" | python3 -c "import json,sys; print(json.load(sys.stdin).get('url',''))")

  # Replace original URL with new URL in /tmp/article.json
  if [ -n "$new_url" ]; then
    python3 -c "
import json
with open('/tmp/article.json') as f: a = json.load(f)
a['body_zh'] = a['body_zh'].replace('$img_url', '$new_url')
with open('/tmp/article.json', 'w') as f: json.dump(a, f, ensure_ascii=False)
"
  fi
done
```

Only do this if `failed_images` is present and non-empty. Skip images that still fail after local download.

### Step 4: Confirm

Tell the user: article title, published date, cover image status, and how many images were embedded.

---

## Command: Batch Import from WeChat

Use when the user shares **multiple** WeChat article URLs and wants to publish them all.

### Step 1: Load API token

```bash
source .env.local
```

### Step 2: Collect all URLs

Ask the user to confirm the list of URLs before starting. Number them 1, 2, 3...

### Step 3: Process each article sequentially

For each URL (index N = 1, 2, 3...):

**3a. Import and save to file immediately**

```bash
curl -s -X POST "https://elsewhere.news/api/import" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"url\": \"WECHAT_URL\"}" > /tmp/import_N.json
cat /tmp/import_N.json | python3 -m json.tool
```

If the import fails (error field in response, or `content` is empty), log the failure and skip to the next URL.

**3b. Reformat Markdown**

Read `/tmp/import_N.json`, apply the same cleanup rules as in "Import from WeChat" Step 3:
- Identify bold paragraphs that are clearly headings → upgrade to `##`
- Remove WeChat embed artifacts
- Remove excessive blank lines

Write the cleaned content back:

```bash
python3 -c "
import json
with open('/tmp/import_N.json') as f: r = json.load(f)
# Apply your cleaned content here
r['content_clean'] = '''CLEANED_CONTENT_HERE'''
with open('/tmp/import_N.json', 'w') as f: json.dump(r, f, ensure_ascii=False)
"
```

**3c. Build and publish**

```bash
python3 -c "
import json, re, time
r = json.load(open('/tmp/import_N.json'))
title = r['title']
content = r.get('content_clean') or r['content']
cover = r.get('cover_image_url', '')
pub = r.get('published_at', '')
excerpt = r.get('excerpt', '')
slug = re.sub(r'[^a-z0-9]+', '-', title.lower())[:50].strip('-') or f'article-{int(time.time())}'
article = {'title_zh': title, 'title_en': 'ENGLISH_TITLE', 'slug': slug,
           'excerpt_zh': excerpt, 'excerpt_en': 'ENGLISH_EXCERPT',
           'body_zh': content, 'cover_image_url': cover, 'published_at': pub}
json.dump(article, open('/tmp/article_N.json', 'w'), ensure_ascii=False)
print('slug:', slug)
"

curl -s -X POST "https://elsewhere.news/api/articles" \
  -H "Authorization: Bearer $ELSEWHERE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d @/tmp/article_N.json | python3 -m json.tool
```

Fill in `ENGLISH_TITLE` and `ENGLISH_EXCERPT` with your own translations before running.

**3d. Handle failed images** (same as single import — only if `failed_images` is present)

**3e. Log result**

After each article, briefly note: ✅ title + URL, or ❌ title + reason for failure. Keep it short to avoid bloating context.

### Step 4: Summary

After all articles are done, show a final table:

| # | 标题 | 状态 | 链接 |
|---|------|------|------|
| 1 | ... | ✅ | ... |
| 2 | ... | ❌ 导入失败 | — |

### Context management note

- Save every import result to `/tmp/import_N.json` immediately — don't hold full article content in memory
- After publishing each article, you only need to remember the title and result URL
- If you feel context is getting long (10+ articles), wrap up the current batch and ask the user if they want to continue in a new session

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
- Always include `title_en` and `excerpt_en` (translate yourself). Never include `body_en` — body translation is handled automatically after publishing
- Always write JSON to temp file and use `curl -d @file` to avoid shell escaping issues
- After registration, human can log into GUI dashboard at `https://elsewhere.news/dashboard/login`
- The API token never expires. If compromised, the human can regenerate it from the dashboard.
