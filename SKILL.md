---
name: content-collector
description: Automatically collect and archive content from shared links in group chats. When a user shares a link (WeChat articles, Feishu docs, web pages, etc.) in any group chat and asks to archive/collect/save it, this skill triggers to fetch the content, create a Feishu document, and update the knowledge base table. Use when: (1) User shares a link and asks to "收录/转存/保存" content, (2) Need to archive web content to Feishu docs, (3) Building a personal knowledge base from shared links, (4) Organizing learning materials from various sources.
---

# Content Collector - 链接内容自动收录技能

## Overview

This skill enables automatic collection and archiving of content from shared links into a structured knowledge base.

**Core Workflow:**
```
Detect Link → Fetch Content → Extract Images → Upload Images to Feishu → Create Feishu Doc → Update Table
```

## When to Use

### 模式1：主动触发（显式关键词）
当用户消息包含以下**触发词**时，立即执行收录：
- "收录" / "转存" / "保存" / "存档" / "存一下" / "归档" / "备份" / "收藏"
- "存到知识库" / "加入知识库" / "转飞书"

**示例：**
- "这个链接收录一下"
- "存到知识库"
- "转存这篇教程"

### 模式2：静默收录（自动检测）
在**群聊场景**中，自动检测以下链接并静默收录：
- 飞书文档/表格/Wiki（feishu.cn）
- 微信公众号文章（mp.weixin.qq.com）
- 技术博客/教程站点
- 知识分享类链接

**静默收录条件：**
1. 消息来自群聊（非私聊）
2. 消息包含可识别的知识类链接
3. 用户没有明确拒绝的意图

**两种模式优先级：**
```
检测到主动触发词 → 立即收录（显式模式）
未检测到触发词但检测到链接 → 静默收录（隐式模式）
```

## Supported Link Types

| Type | Example | Fetch Method |
|------|---------|--------------|
| WeChat Article | `https://mp.weixin.qq.com/s/xxx` | kimi_fetch |
| Feishu Doc | `https://xxx.feishu.cn/docx/xxx` | feishu_fetch_doc |
| Feishu Wiki | `https://xxx.feishu.cn/wiki/xxx` | feishu_fetch_doc |
| Web Page | General URLs | kimi_fetch / web_fetch |

## Supported Image Sources

| Source | Example | Priority | Notes |
|--------|---------|----------|-------|
| Markdown image | `![alt](https://xx/image.png)` | High | 直接替换为飞书 image_key |
| HTML `<img src>` | `<img src="/assets/a.png">` | High | 相对路径需转绝对路径 |
| Lazy-load image | `data-src`, `data-original` | Medium | 常见于公众号/博客懒加载 |
| `srcset` candidate | `srcset="a 1x, b 2x"` | Medium | 优先选择清晰度更高的候选图 |
| Feishu file token | `boxcn...` / `img_v3_...` | High | 需要走飞书素材下载后再上传 |

## Global Availability (全局可用配置)

**生效范围：所有用户、所有群聊**

本技能已配置为全局可用，支持以下对象：

| 对象类型 | 支持状态 | 说明 |
|---------|---------|------|
| **所有用户** | ✅ 可用 | 任何用户分享的链接均可被收录 |
| **所有群聊** | ✅ 可用 | 支持技能中心群、养虾群、学习群等所有群组 |
| **私聊消息** | ✅ 可用 | 用户私信分享链接也可触发收录 |
| **多渠道** | ✅ 可用 | 飞书、其他渠道统一支持 |

**权限说明：**
- 任何用户均可触发收录（无需管理员权限）
- 收录的文档统一存储到指定的知识库目录
- 所有用户均可查看已收录的文档

---

## Installation & Permission Check (安装与权限检查)

在正式使用本技能前，系统必须自动或引导用户完成以下权限校验，以确保流程不中断：

### 1. 飞书权限清单
| 权限项 | 验证工具 | 目的 |
|-------|---------|------|
| **OAuth 授权** | `feishu_oauth` | 获取操作飞书文档和表格的用户凭证 |
| **知识库写入权限** | `feishu_create_doc` | 确保能在指定的 Space ID 下创建节点 |
| **多维表格编辑权限** | `feishu_bitable_app_table_record` | 确保能向指定的 app_token 写入记录 |
| **图片上传权限** | `feishu_im_bot_upload` | 允许将本地图片同步至飞书素材库 |

### 2. 预检流程 (Pre-flight Check)
每次“安装”或配置更新后，执行以下检查：
1. **验证 Space ID 可访问性**：尝试在指定目录下获取节点列表。
2. **验证 Table 结构**：检查 `关键词`、`原链接`、`图片数量`、`图片处理状态` 等字段是否存在（后两者可选）。
3. **静默测试**：如果权限不足，立即通过 `feishu_oauth` 弹出授权引导，而非在执行收录时报错。

---

## Configuration

Before using, ensure these are configured in MEMORY.md:

```markdown
## Content Collector Config
- **Knowledge Base Table**: `[Your Bitable App Token]` (Bitable app_token)
- **Table URL**: [Your Bitable Table URL]
- **Default Table ID**: `[Your Table ID]` (will auto-detect if available)
- **Knowledge Base Space ID**: `[Your Space ID]` (所有文档创建在此知识库下)
- **Knowledge Base URL**: [Your Knowledge Base Homepage URL]
- **Content Categories**: 技术教程, 实战案例, 产品文档, 学习笔记
- **Global Access**: 所有用户可用，所有群聊可用
- **Image Fetch Mode**: `all` / `cover_only`（默认 `all`）
- **Image Max Count**: `20`（单篇文档最多处理图片数）
- **Image Max Size MB**: `10`（单图超过阈值则跳过）
- **Image Timeout Sec**: `20`（下载超时）
- **Image Allowed Types**: `jpg,png,gif,webp`
- **Image Fallback**: `keep_original_link=true`
```

**Note**: 
1. This skill updates ONLY the configured knowledge base table. Do not create or update any other tables.
2. **All created documents must be saved under the designated Knowledge Base** using wiki_node parameter.
3. **Global Access**: 所有用户、所有群聊均可使用本技能，收录的文档对全员可见。
4. 图片抓取默认开启；若用户明确要求“纯文字收录”，可跳过图片处理。

---

## 📚 知识库文档存储规则（必遵守）

所有收录的文档必须按照以下规则分类存储到知识库对应目录：

### 知识库目录结构

请参考各项目或团队定义的知识库标准目录结构进行存储。收录的文档通常存放在“素材”或“归档”类目录下。

### 文档分类映射规则

| 内容分类 | 存储目录 (wiki_node) | 命名前缀 | 示例 |
|----------|---------------------|----------|------|
| 技术教程 | `F9pFw9dxTiXmpsk5bNlco704nag` (内容文档) | 📖 | 📖 [标题] |
| 实战案例 | `F9pFw9dxTiXmpsk5bNlco704nag` (内容文档) | 🛠️ | 🛠️ [标题] |
| 产品文档 | `F9pFw9dxTiXmpsk5bNlco704nag` (内容文档) | 📄 | 📄 [标题] |
| 学习笔记 | `F9pFw9dxTiXmpsk5bNlco704nag` (内容文档) | 💡 | 💡 [标题] |
| 热点资讯 | `F9pFw9dxTiXmpsk5bNlco704nag` (内容文档) | 🔥 | 🔥 [标题] |
| 设计技能 | `F9pFw9dxTiXmpsk5bNlco704nag` (内容文档) | 🎨 | 🎨 [标题] |
| 工具推荐 | `F9pFw9dxTiXmpsk5bNlco704nag` (内容文档) | 🔧 | 🔧 [标题] |
| 训练营 | `F9pFw9dxTiXmpsk5bNlco704nag` (内容文档) | 🎓 | 🎓 [标题] |

### 文档命名规范

```
[Emoji前缀] [原标题] | 收录日期

示例：
📖 OpenClaw保姆级教程 | 2026-03-08
🛠️ 火山方舟自动化报表案例 | 2026-03-08
🔥 GPT-5.4发布解读 | 2026-03-08
```

### 文档模板

```markdown
# [Emoji] [原标题]

> 📌 **元信息**
> - 来源：[原始来源]
> - 原文链接：[原始URL]
> - 收录时间：YYYY-MM-DD
> - 内容分类：[技术教程/实战案例/产品文档/学习笔记/热点资讯/设计技能/工具推荐/训练营]
> - 关键词：[关键词1, 关键词2, 关键词3]

---

## 📋 核心要点

[3-5条核心内容摘要]

---

## 📝 正文内容

[完整的转存内容]

---

## 🔗 相关链接

- 原文链接：[原始URL]
- 知识库索引：[素材池文档索引链接]

---

📚 **收录时间**：YYYY-MM-DD  
🏷️ **分类**：[分类名]  
🔖 **关键词**：[关键词]
```

### 自动更新素材索引

每次收录完成后，必须：

1. **更新多维表格** - 添加新记录到素材池表格
2. **更新素材索引文档** - 在「📚 内容素材池文档索引」中添加条目
3. **更新分类统计** - 更新各分类的文档数量和占比

---

## Workflow

### Step 1: Detect and Parse Link

Extract URL from user message using regex or direct extraction.

### Step 2: Fetch Content (正文 + 原始结构)

Choose appropriate fetch method based on URL pattern:

**For WeChat articles:**
```python
raw = kimi_fetch(url="https://mp.weixin.qq.com/s/xxx")
```

**For Feishu docs:**
```python
raw = feishu_fetch_doc(doc_id="https://xxx.feishu.cn/docx/xxx")
```

**For general web pages:**
```python
raw = kimi_fetch(url="https://example.com/article")
# or
raw = web_fetch(url="https://example.com/article")
```

**Standardized Output (必须统一):**
```python
fetched = {
    "title": raw.get("title", ""),
    "markdown": raw.get("markdown", raw.get("content", "")),
    "raw_html": raw.get("html", ""),
    "source_url": original_url
}
```

### Step 3: Analyze and Categorize

**智能分类判断：**
根据内容特征自动判断分类：

| 判断依据 | 分类 |
|----------|------|
| 包含"安装/配置/部署/教程"等词 | 📖 技术教程 |
| 包含"案例/实战/项目/演示"等词 | 🛠️ 实战案例 |
| 包含"安全/公告/版本/功能"等词 | 📄 产品文档 |
| 包含"学习/成长/指南/笔记"等词 | 💡 学习笔记 |
| 包含"发布/新功能/热点"等词 | 🔥 热点资讯 |
| 包含"设计/Prompt/美学"等词 | 🎨 设计技能 |
| 包含"工具/CLI/插件"等词 | 🔧 工具推荐 |
| 包含"训练营/课程/教学"等词 | 🎓 训练营 |

### Step 4: Process Images (图片处理)

在创建飞书文档前，必须执行图片抓取与回填，目标是“最大化保留原文图片、最小化失败影响正文”。

**Image Processing Workflow v2:**
```python
import os
import re
from urllib.parse import urljoin

IMG_MD_RE = re.compile(r'!\[(.*?)\]\(([^)]+)\)')
IMG_HTML_RE = re.compile(r'<img[^>]+(?:src|data-src|data-original)=["\']([^"\']+)["\']', re.I)
IMG_SRCSET_RE = re.compile(r'<img[^>]+srcset=["\']([^"\']+)["\']', re.I)

def normalize_img_ref(ref: str, base_url: str) -> str:
    ref = (ref or "").strip()
    if not ref:
        return ""
    if ref.startswith(("http://", "https://")):
        return ref
    if ref.startswith("//"):
        return "https:" + ref
    # 飞书 token 或相对路径
    if ref.startswith(("img_v3_", "boxcn", "file_", "AAM")):
        return ref
    return urljoin(base_url, ref)

def pick_srcset_candidate(srcset_value: str) -> str:
    # 示例: "a.jpg 1x, b.jpg 2x" 或 "a.jpg 480w, b.jpg 1080w"
    parts = [x.strip() for x in (srcset_value or "").split(",") if x.strip()]
    if not parts:
        return ""
    return parts[-1].split(" ")[0].strip()

def extract_image_candidates(markdown_content: str, raw_html: str, source_url: str):
    candidates = []
    for alt, ref in IMG_MD_RE.findall(markdown_content or ""):
        candidates.append({"alt": alt or "image", "ref": normalize_img_ref(ref, source_url), "from_md": True})
    for ref in IMG_HTML_RE.findall(raw_html or ""):
        normalized = normalize_img_ref(ref, source_url)
        if normalized:
            candidates.append({"alt": "image", "ref": normalized, "from_md": False})
    for srcset_value in IMG_SRCSET_RE.findall(raw_html or ""):
        candidate = normalize_img_ref(pick_srcset_candidate(srcset_value), source_url)
        if candidate:
            candidates.append({"alt": "image", "ref": candidate, "from_md": False})
    # 去重，保持首次出现顺序
    seen, ordered = set(), []
    for item in candidates:
        if item["ref"] and item["ref"] not in seen:
            seen.add(item["ref"])
            ordered.append(item)
    return ordered

def fetch_and_upload_images(markdown_content, raw_html, source_url, cfg):
    candidates = extract_image_candidates(markdown_content, raw_html, source_url)
    max_count = int(cfg.get("image_max_count", 20))
    max_bytes = int(cfg.get("image_max_size_mb", 10)) * 1024 * 1024

    replace_map = {}
    uploaded_extra = []
    failed = []
    total = 0
    success = 0

    for item in candidates[:max_count]:
        ref = item["ref"]
        total += 1
        tmp_path = None
        try:
            # 1) 下载图片到本地临时文件
            if ref.startswith(("http://", "https://")):
                tmp_path = download_image_to_local(
                    ref,
                    timeout=int(cfg.get("image_timeout_sec", 20)),
                    max_bytes=max_bytes,
                    allowed_types=cfg.get("image_allowed_types", "jpg,png,gif,webp")
                )
            else:
                # 飞书 file_token / image_key
                tmp_path = download_feishu_media_to_local(
                    ref,
                    max_bytes=max_bytes,
                    allowed_types=cfg.get("image_allowed_types", "jpg,png,gif,webp")
                )

            # 2) 上传到飞书素材
            upload_result = feishu_im_bot_upload(action="upload_image", file_path=tmp_path)
            image_key = upload_result.get("image_key")
            if not image_key:
                raise RuntimeError("empty image_key")

            success += 1
            replace_map[ref] = image_key
            if not item["from_md"]:
                uploaded_extra.append((item["alt"], image_key))
        except Exception as e:
            failed.append({"ref": ref, "reason": str(e)[:120]})
        finally:
            if tmp_path and os.path.exists(tmp_path):
                os.remove(tmp_path)

    # 3) 回填 markdown 中已有图片
    processed = markdown_content
    for src, image_key in replace_map.items():
        processed = processed.replace(f"({src})", f"({image_key})")

    # 4) 将 HTML-only 图片追加到文末，避免丢图
    if uploaded_extra:
        lines = ["", "---", "", "## 🖼️ 原文配图（自动抓取）", ""]
        for alt, image_key in uploaded_extra:
            lines.append(f"![{alt}]({image_key})")
        processed += "\n".join(lines)

    # 5) 失败兜底提示（不阻断收录）
    if failed:
        processed += (
            "\n\n> ⚠️ 部分图片处理失败，已保留正文收录。"
            f" 成功 {success}/{total}，失败 {len(failed)}。"
        )

    image_stats = {
        "total": total,
        "success": success,
        "failed": len(failed),
        "failed_refs": failed
    }
    return processed, image_stats
```

**Fallback Strategy:**
- 单张图片失败不影响整篇文档收录
- 原文中存在但未成功托管的图片，保留原链接并记录失败原因
- 文档末尾自动追加“原文配图/失败提示”区块，确保信息不丢失

### Step 5: Create Feishu Document (按知识库规则存储)

Convert processed markdown to Feishu document with proper organization:

```python
# 0. 先执行图片处理
processed_markdown_content, image_stats = fetch_and_upload_images(
    markdown_content=fetched["markdown"],
    raw_html=fetched["raw_html"],
    source_url=fetched["source_url"],
    cfg=config
)

# 1. 确定分类和参数
content_category = classify_content(processed_markdown_content)  # 📖/🛠️/📄/💡/🔥/🎨/🔧/🎓
emoji_prefix = get_emoji_prefix(content_category)  # 根据分类获取emoji
wiki_node = get_wiki_node_by_category(content_category)  # 获取存储目录

# 2. 生成文档标题
doc_title = f"{emoji_prefix} {original_title} | {today_date}"

# 3. 生成文档内容（使用标准模板）
doc_content = f"""# {emoji_prefix} {original_title}

> 📌 **元信息**
> - 来源：{source_name}
> - 原文链接：{original_url}
> - 收录时间：{today_date}
> - 内容分类：{content_category}
> - 关键词：{keywords}
> - 图片处理：成功 {image_stats["success"]}/{image_stats["total"]}（失败 {image_stats["failed"]}）

---

## 📋 核心要点

{extract_key_points(processed_markdown_content, 5)}

---

## 📝 正文内容

{processed_markdown_content}

---

## 🔗 相关链接

- 原文链接：{original_url}
- 知识库索引：[Your Index Document URL]

---

📅 **收录时间**：{today_date}  
🏷️ **分类**：{content_category}  
🔖 **关键词**：{keywords}
"""

# 4. 创建文档到知识库对应目录
feishu_create_doc(
    title=doc_title,
    markdown=doc_content,
    wiki_node=wiki_node  # 必须指定存储目录
)
```

**存储目录映射：**
| 分类 | wiki_node | 目录名 |
|------|-----------|--------|
| 所有素材 | `F9pFw9dxTiXmpsk5bNlco704nag` | 04-内容素材 |

**IMPORTANT**: 
1. All documents MUST be created under the designated Knowledge Base using wiki_node parameter.
2. Documents must follow the naming convention: `[Emoji] [Title] | [Date]`
3. Documents must use the standard template with metadata section.

### Step 6: Update Knowledge Base Table

Add record to the Bitable knowledge base (ONLY update this specific table):

```python
feishu_bitable_app_table_record(
    action="create",
    app_token="[Your App Token]",  # Configured in MEMORY.md
    table_id="[Your Table ID]",  # Will use correct table ID from the base
    fields={
        "关键词": keywords,
        "内容分类": content_category,
        "文档标题": [{"text": original_title, "type": "text"}],
        "来源": [{"text": source_name, "type": "text"}],
        "核心要点": [{"text": key_points, "type": "text"}],
        "飞书文档链接": {"link": new_doc_url, "text": "飞书文档", "type": "url"},
        "原链接": {"link": original_url, "text": "原文链接", "type": "url"},
        "图片数量": image_stats["total"],
        "图片处理状态": f'{image_stats["success"]}/{image_stats["total"]} 成功',
        "图片失败数": image_stats["failed"]
    }
)
```

**Table Fields:**
| Field | Type | Description |
|-------|------|-------------|
| 关键词 | Text | Search keywords for the content |
| 内容分类 | Single Select | Category: 📖技术教程/🛠️实战案例/📄产品文档/💡学习笔记/🔥热点资讯/🎨设计技能/🔧工具推荐/🎓训练营 |
| 文档标题 | Text | Title of the archived document |
| 来源 | Text | Original source name |
| 核心要点 | Text | Key points summary (3-5 items) |
| 飞书文档链接 | URL | Link to the created Feishu document |
| 原链接 | URL | **Original source URL** - 新增字段，存储采集的原始链接 |
| 图片数量 | Number | 本次检测到的图片总数 |
| 图片处理状态 | Text | 图片托管结果，例如 `8/10 成功` |
| 图片失败数 | Number | 上传失败图片数，用于质量监控 |

**IMPORTANT**: Only update the configured knowledge base table. Never create or modify other tables.

### Step 7: Update Content Index Document

After creating the document and updating the table, MUST update the index document:

```python
# 1. 获取当前索引文档内容
index_doc = feishu_fetch_doc(doc_id="[Your Index Doc ID]")

# 2. 在对应分类表格中添加新行
new_index_entry = f"| {original_title} | {source_name} | [查看]({new_doc_url}) |\n"

# 3. 更新分类统计
update_category_stats(content_category)

# 4. 更新总计数
update_total_count()
```

**或者直接追加到索引文档的末尾：**
```python
feishu_update_doc(
    doc_id="[Your Index Doc ID]",
    mode="append",
    markdown=f"""
| {original_title} | {source_name} | [查看]({new_doc_url}) |
"""
)
```

---

## Content Categorization Guide

| Category | Emoji | Description | Examples |
|----------|-------|-------------|----------|
| **技术教程** | 📖 | Step-by-step technical guides | Installation, configuration, API usage |
| **实战案例** | 🛠️ | Real-world implementation examples | Case studies, project demos |
| **产品文档** | 📄 | Product features, security notices | Release notes, security advisories |
| **学习笔记** | 💡 | Conceptual knowledge, methodologies | Best practices, architecture guides |
| **热点资讯** | 🔥 | Breaking news, releases | GPT-5.4, new features |
| **设计技能** | 🎨 | Design, prompts, aesthetics | AJ's prompts, design guides |
| **工具推荐** | 🔧 | Tools, CLI, plugins | gws, trae, autotools |
| **训练营** | 🎓 | Courses, bootcamps, tutorials | OpenClaw bootcamp |

**分类判断优先级：**
1. 优先根据用户指定分类
2. 其次根据标题关键词
3. 最后根据内容特征自动判断
4. 不确定时标记为"待分类"，请用户确认

## Delete Record Process

When user replies "删除" or "删除 [keyword]":

```python
# 1. Search records by keyword
feishu_bitable_app_table_record(
    action="list",
    app_token="[Your App Token]",
    table_id="[Your Table ID]",
    filter={
        "conjunction": "and",
        "conditions": [
            {"field_name": "关键词", "operator": "contains", "value": [keyword]}
        ]
    }
)

# 2. Confirm deletion
# If multiple found → list for user to select
# If single found → ask for confirmation

# 3. Execute deletion
feishu_bitable_app_table_record(
    action="delete",
    app_token="[Your App Token]",
    table_id="[Your Table ID]",
    record_id="record_id_to_delete"
)
```

## Error Handling

### Common Issues

| Error | Cause | Solution |
|-------|-------|----------|
| Fetch timeout | Network issue or heavy content | Retry with longer timeout, or use alternative fetch method |
| Unauthenticated | OAuth token expired or not authed | Trigger `feishu_oauth` to refresh user credentials |
| Permission denied | No write access to Space/Table | Check if user/bot has 'Editor' role in Feishu |
| Content too long | Exceeds API limits | Truncate or split into multiple documents |
| Table update failed | Wrong app_token or table_id | Verify configuration in MEMORY.md |
| Field Missing | "原链接" field not in table | Add the field to Bitable manually or via API |
| Image download failed | Source anti-hotlinking / timeout | Retry with headers, then keep original link |
| Image too large | Exceeds size limit | Compress or skip and log warning |
| Invalid image type | Unsupported format or broken file | Skip image and continue document creation |

### Recovery Steps

1. If fetch fails → Try alternative method (kimi_fetch → web_fetch)
2. If image fetch/upload fails → Keep original image link and append warning block
3. If Feishu doc creation fails → Check OAuth status
4. If table update fails → Verify table structure and field names
5. Always report partial success (doc created but table not updated)

## Response Template

### 收录成功响应（流式Post格式）

```json
{
  "msg_type": "post",
  "content": {
    "post": {
      "zh_cn": {
        "title": "✅ 收录完成",
        "content": [
          [
            {"tag": "text", "text": "📄 "},
            {"tag": "text", "text": "{emoji} {原标题} | {日期}", "style": {"bold": true}}
          ],
          [{"tag": "text", "text": ""}],
          [
            {"tag": "text", "text": "💡 文档亮点：", "style": {"bold": true}}
          ],
          [
            {"tag": "text", "text": "• {亮点1}"}
          ],
          [
            {"tag": "text", "text": "• {亮点2}"}
          ],
          [
            {"tag": "text", "text": "• {亮点3}"}
          ],
          [{"tag": "text", "text": ""}],
          [
            {"tag": "text", "text": "🔗 "},
            {"tag": "a", "text": "查看飞书文档", "href": "{文档URL}"}
          ]
        ]
      }
    }
  }
}
```

**简洁输出示例：**
```
✅ 收录完成

📄 📖 OpenClaw配置指南 | 2026-03-08

💡 文档亮点：
• 完整配置示例，含9大模块详解
• 多Agent扩展配置方案
• 生产环境安全配置建议

🔗 查看飞书文档 → [点击打开](https://xxx.feishu.cn/docx/xxx)
```

### 静默收录响应（流式Post格式）

```json
{
  "msg_type": "post",
  "content": {
    "post": {
      "zh_cn": {
        "title": "✅ 已自动收录",
        "content": [
          [
            {"tag": "text", "text": "📄 "},
            {"tag": "text", "text": "{emoji} {原标题}", "style": {"bold": true}}
          ],
          [{"tag": "text", "text": ""}],
          [
            {"tag": "text", "text": "💡 亮点：{亮点摘要}"}
          ],
          [{"tag": "text", "text": ""}],
          [
            {"tag": "a", "text": "📎 查看文档", "href": "{文档URL}"}
          ]
        ]
      }
    }
  }
}
```

### 批量收录响应（流式Post格式）

```json
{
  "msg_type": "post",
  "content": {
    "post": {
      "zh_cn": {
        "title": "✅ 批量收录完成（{N}份）",
        "content": [
          [
            {"tag": "text", "text": "📄 {emoji1} {标题1}", "style": {"bold": true}}
          ],
          [
            {"tag": "text", "text": "   💡 {亮点1}"}
          ],
          [
            {"tag": "a", "text": "   🔗 查看", "href": "{链接1}"}
          ],
          [{"tag": "text", "text": ""}],
          [
            {"tag": "text", "text": "📄 {emoji2} {标题2}", "style": {"bold": true}}
          ],
          [
            {"tag": "text", "text": "   💡 {亮点2}"}
          ],
          [
            {"tag": "a", "text": "   🔗 查看", "href": "{链接2}"}
          ]
        ]
      }
    }
  }
}
```

**输出原则：**
1. **必须流式Post格式** - 使用 msg_type: post
2. **只包含3个核心要素：**
   - 文件名称（📄 Emoji + 标题 + 日期）
   - 文档亮点（💡 3-5条核心要点）
   - 飞书链接（🔗 点击查看）
3. **不输出其他信息** - 不显示分类、不显示表格更新、不显示统计
4. **保持简洁** - 每份文档3-5行内容

## Best Practices

1. **Always verify content was fetched correctly** before creating documents
2. **Extract key insights** from the content for the summary
3. **Use appropriate category** based on content nature
4. **Generate relevant keywords** for better searchability
5. **Keep source attribution** clear for copyright respect
6. **Handle partial failures gracefully** - document what succeeded and what failed
7. **Update index document** - Every new document must be added to the index
8. **Follow naming convention** - Use [Emoji] [Title] | [Date] format
9. **Store in correct directory** - Use wiki_node to place in right category
10. **Image-first fallback** - 图片失败不阻断正文入库，优先保证知识沉淀完整性

## 收录完成检查清单 (Checklist)

每次收录必须完成以下所有步骤：

- [ ] **执行权限预检**（验证 OAuth 及 Space/Table 写入权限）
- [ ] 获取并处理原始内容（含图片）
- [ ] 抽取并去重图片引用（Markdown + HTML）
- [ ] 图片托管到飞书（记录总数、成功数、失败数）
- [ ] 智能分类并确定 Emoji 前缀
- [ ] 提取核心要点（3-5条）
- [ ] 生成关键词
- [ ] **创建飞书文档**（使用标准模板，指定 wiki_node）
- [ ] **更新多维表格**（添加完整记录，包含**原链接/图片统计**字段）
- [ ] **更新文档索引**（在素材索引中添加条目）
- [ ] 发送收录完成通知给用户

**任何一步未完成，视为收录失败！**

## Integration with Memory

After each collection, update MEMORY.md:

```markdown
### YYYY-MM-DD - Content Collection
- **新增收录**: [Title]
- **来源**: [Source]
- **分类**: [Category]
- **知识库状态**: 共[N]条记录
- **索引更新**: ✅ 已更新
```

This skill is part of the core knowledge management system. Execute with care and attention to detail.

---

## 附录：图片抓取能力 v2（执行约束）

### 必须满足的目标
1. **不丢图**：Markdown 图片 + HTML 图片都要尝试收录。
2. **不阻断**：图片失败不能阻断正文文档创建。
3. **可观测**：表格必须记录图片处理统计（总数/成功/失败）。
4. **可回溯**：失败图片需保留原始引用，便于后续补抓。

### 推荐执行策略
1. 先用 `extract_image_candidates` 统一收集并去重。
2. 每篇文档最多处理 `image_max_count` 张，防止超时。
3. 单图限制 `image_max_size_mb`，超过阈值直接跳过并计入失败。
4. 仅允许常见格式（jpg/png/gif/webp），其余按失败处理。
5. 所有临时文件在 `finally` 中删除，避免磁盘残留。

### 最小可行验收标准（MVP）
- 输入含 10 张图的公众号文章，最终文档中至少 8 张能正常显示。
- 即使图片全部失败，也必须产出正文文档并回写表格记录。
- 收录响应中必须返回飞书文档链接，且不暴露内部异常堆栈。

---

*图片处理方案 v2.0 - 2026-03-17*
