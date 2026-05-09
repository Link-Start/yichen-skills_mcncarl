---
name: x-article-draft-uploader
description: 将 Obsidian 或本地 Markdown 文章上传到 X/Twitter Articles 草稿，自动把第一张图作为封面，并按原文位置插入所有正文图片。适用于用户要求上传、发布、保存 Markdown 到 X Article，尤其是需要复用 Chrome 登录态、使用独立 Playwright 浏览器、不接管用户当前浏览器、封面必须是最上方图片，或旧脚本出现缺图、错位、MPH_MARKER 等残留时。
---

# X Article Draft Uploader

## 核心原则

只创建草稿。除非用户明确确认可以公开发布，否则不要点击 X 最终的 `发布` 按钮。

使用独立的 Playwright 浏览器会话，不要抢占用户当前正在用的 Chrome 窗口。可以复用 Chrome 登录态，但只通过临时导出的 Playwright cookie JSON 使用。

## 快速开始

1. 如果 `/tmp/x_current_cookies.json` 不存在或登录态失效，先导出当前 X cookies：

```bash
python3 ~/.codex/skills/x-article-draft-uploader/scripts/export_x_cookies_from_chrome.py --output /tmp/x_current_cookies.json
```

2. 正式碰 X 之前，先 dry-run 解析文章，确认封面、正文图数量和锚点：

```bash
python3 ~/.codex/skills/x-article-draft-uploader/scripts/upload_markdown_to_x_article.py \
  "/absolute/path/to/article.md" \
  --cookies-json /tmp/x_current_cookies.json \
  --dry-run
```

3. 新建一篇干净的 X Article 草稿并上传：

```bash
python3 ~/.codex/skills/x-article-draft-uploader/scripts/upload_markdown_to_x_article.py \
  "/absolute/path/to/article.md" \
  --cookies-json /tmp/x_current_cookies.json
```

脚本会输出这些结果文件：

- 草稿 URL：`/tmp/x_article_upload_url.txt`
- 校验结果 JSON：`/tmp/x_article_upload_result.json`
- 最终截图：`/tmp/x_article_final_uploaded.png`

## 操作流程

1. 使用本 Skill 自带的 `scripts/parse_markdown.py` 解析 Markdown。
2. 把文章里的第一张图片当作封面图。
3. 从原始 Markdown 中读取每张正文图前一行，作为图片插入锚点。列表场景尤其要注意：用真实的上一行，比如 `Git 变化。`，不要用前面列表项拼出来的更长 fallback。
4. 启动全新的 Playwright Chromium context，并添加 X cookies。
5. 打开 `https://x.com/compose/articles`，点击 `create`，记录新生成的 `/compose/articles/edit/...` URL。
6. 通过封面区域的 file input 上传封面，然后点击 X 的 `应用`。如果跳过这步，X 会留下 media-edit mask 挡住编辑器，而且封面不会真正保存。
7. 填写标题，并把 rich HTML 正文粘贴到 `[data-testid="composer"]`。
8. 正文图片从后往前插入。每张图都按这个顺序处理：
   - 在当前编辑器里找到优先级最高的 anchor。
   - 点击该段落末尾。
   - 按 `End`，再按 `Enter`。
   - 通过 clipboard paste event 粘贴图片文件。
   - 等页面里检测到的 media count 增加后再继续下一张。
9. 等 X autosave 完成，然后校验标题、正文开头/结尾、没有 `MPH_MARKER`，并确认总媒体数等于 `1 + expected_image_count`。

## 常见卡点

- 如果 X 跳转到 `/login`，说明登录态失效，重新导出 cookies。
- 如果上传封面后编辑器被遮罩挡住，找 `应用` 按钮并点击。
- 正文图片不要直接用隐藏 file input。它可能指向封面上传器，导致替换封面。
- 不要只看 media count。列表后的图片尤其要看 anchor 是否正确。脚本会在结果 JSON 里记录 `anchor_used` 和 `expected_anchor`。
- 如果某次运行只是部分成功，但图片位置不对，优先新建干净草稿重跑，不要在旧草稿上硬修。

## 脚本说明

- `upload_markdown_to_x_article.py --dry-run` 是安全检查，不会打开 X。
- 上传脚本需要 Python Playwright 和有效的 X cookie JSON。
- Markdown parser 已经内置在当前 Skill 里，不依赖旧的 `x-article-publisher` Skill。
- cookie 导出脚本只读取本机 Chrome cookies，并写入指定的临时 Playwright cookie 文件；Skill 本身不保存任何 cookie。
