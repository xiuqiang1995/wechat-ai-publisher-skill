---
name: wechat-ai-publisher
description: 发布文章到微信公众号草稿箱。当用户说"发布公众号"、"发布到微信"、"publish wechat"时触发。
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
  - WebFetch
  - WebSearch
  - mcp__exa__web_search_exa
---


# 微信公众号发布工具

## 执行流程

```
1. 询问用户配置（风格、配图数量）
       ↓
2. 采集素材（WebSearch/Exa）
       ↓
3. 撰写文章（Markdown）
       ↓
4. 生成配图（Replicate API）
       ↓
5. 发布到公众号草稿箱
```

## Step 1: 询问用户配置（必须执行）

使用 AskUserQuestion 工具询问用户：

**问题 1 - CSS 风格**：
- 紫色经典 purple（推荐）
- 橙心暖色 orangeheart
- GitHub风格 github

**问题 2 - 配图数量**：
- 仅封面图
- 封面 + 2张配图
- 封面 + 3张配图（推荐）

## Step 2-5: 发布脚本

```bash
# 完整流程（生成图片 + 转换 + 发布）
source ~/.zshrc && /opt/homebrew/bin/python3.12 \
  ~/.claude/skills/wechat-ai-publisher/scripts/wechat_publisher.py \
  --markdown /tmp/{article_name}.md \
  --style {用户选择的风格} \
  --images {用户选择的配图数量}

# 跳过图片生成（使用已有图片）
source ~/.zshrc && /opt/homebrew/bin/python3.12 \
  ~/.claude/skills/wechat-ai-publisher/scripts/wechat_publisher.py \
  --markdown /tmp/{article_name}.md \
  --skip-images
```

## 参数说明

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--markdown, -m` | Markdown 文件路径 | 必填 |
| `--style, -s` | CSS 风格 (purple/orangeheart/github) | purple |
| `--images, -i` | 配图数量 (0-3) | 3 |
| `--title, -t` | 文章标题 | 从 Markdown 提取 |
| `--digest, -d` | 文章摘要 | 从 Markdown 提取 |
| `--output-dir, -o` | 输出目录 | /tmp/wechat_publish |
| `--skip-preflight` | 跳过环境预检 | false |
| `--skip-images` | 跳过图片生成 | false |

## CSS 风格选项

| 风格 ID | 名称 | 主色调 | 适用场景 |
|---------|------|--------|----------|
| `purple` | 紫色经典 | #8064a9 | 科技深度文章（默认） |
| `orangeheart` | 橙心暖色 | #ef7060 | 热点解读 |
| `github` | GitHub风格 | #333333 | 代码教程 |

## Markdown 格式要求

```markdown
# 文章标题

开头段落...

## 第一部分

内容...

<!-- IMAGE_1: 配图1的描述，用于 AI 生成 -->

## 第二部分

内容...

<!-- IMAGE_2: 配图2的描述 -->

## 第三部分

内容...

<!-- IMAGE_3: 配图3的描述 -->
```

## 环境变量

必须在 `~/.zshrc` 中配置：

```bash
export WECHAT_APPID="your_appid"
export WECHAT_SECRET="your_secret"
export REPLICATE_API_TOKEN="your_token"
```

2. **图片生成失败时立即停止**：如果 Replicate API 返回错误或网络问题导致图片生成失败，立即告知用户并停止流程，不要使用 `--skip-images` 强行继续（会导致使用旧的缓存图片）。

3. **错误处理**：遇到任何错误，向用户报告具体错误信息，等待用户决定下一步操作。

