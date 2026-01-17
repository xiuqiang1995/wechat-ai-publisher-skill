#!/opt/homebrew/bin/python3.12
"""
微信公众号文章发布工具 - 全流程自动化

用法：
  python wechat_publisher.py --markdown article.md --style purple --images 3
  python wechat_publisher.py --markdown article.md  # 使用默认配置

环境变量：
  WECHAT_APPID        - 公众号 AppID
  WECHAT_SECRET       - 公众号 AppSecret
  REPLICATE_API_TOKEN - Replicate API token
"""

import argparse
import json
import os
import re
import subprocess
import sys
from pathlib import Path

# ============================================================================
# 配置
# ============================================================================

PYTHON_BIN = "/opt/homebrew/bin/python3.12"
GENERATE_IMAGE_SCRIPT = Path.home() / ".claude/skills/article-illustrator/scripts/generate_image.py"

STYLES = {
    "purple": {
        "title_color": "#8064a9",
        "text_color": "#444444",
        "quote_bg": "#f4f2f9",
        "code_bg": "#f6f8fa",
        "code_color": "#24292e",
        "font": "Open Sans, -apple-system, BlinkMacSystemFont, sans-serif"
    },
    "orangeheart": {
        "title_color": "#ef7060",
        "text_color": "#000000",
        "quote_bg": "#fff5f5",
        "code_bg": "#f6f8fa",
        "code_color": "#24292e",
        "font": "Optima, -apple-system, BlinkMacSystemFont, sans-serif"
    },
    "github": {
        "title_color": "#333333",
        "text_color": "#333333",
        "quote_bg": "#f6f8fa",
        "code_bg": "#f6f8fa",
        "code_color": "#24292e",
        "font": "Open Sans, -apple-system, BlinkMacSystemFont, sans-serif"
    }
}

# ============================================================================
# 预检
# ============================================================================

def preflight() -> bool:
    """环境预检，返回 True 表示通过"""
    errors = []
    
    # 检查 Python 版本
    if sys.version_info < (3, 10) or sys.version_info >= (3, 14):
        errors.append(f"Python 版本不兼容: {sys.version_info.major}.{sys.version_info.minor} (需要 3.10-3.13)")
    
    # 检查 replicate
    try:
        import replicate
    except ImportError:
        errors.append("缺少 replicate 库，运行: pip install replicate")
    
    # 检查 requests
    try:
        import requests
    except ImportError:
        errors.append("缺少 requests 库，运行: pip install requests")
    
    # 检查环境变量
    if not os.environ.get("REPLICATE_API_TOKEN"):
        errors.append("缺少环境变量: REPLICATE_API_TOKEN")
    if not os.environ.get("WECHAT_APPID"):
        errors.append("缺少环境变量: WECHAT_APPID")
    if not os.environ.get("WECHAT_SECRET"):
        errors.append("缺少环境变量: WECHAT_SECRET")
    
    # 检查 ImageMagick
    result = subprocess.run(["which", "magick"], capture_output=True)
    if result.returncode != 0:
        errors.append("缺少 ImageMagick，运行: brew install imagemagick")
    
    # 检查 generate_image.py
    if not GENERATE_IMAGE_SCRIPT.exists():
        errors.append(f"缺少图片生成脚本: {GENERATE_IMAGE_SCRIPT}")
    
    if errors:
        print("=" * 60)
        print("预检失败")
        print("=" * 60)
        for e in errors:
            print(f"  ✗ {e}")
        print()
        return False
    
    print("预检通过 ✓")
    return True

# ============================================================================
# 图片生成
# ============================================================================

def generate_images(prompts: dict, output_dir: Path) -> dict:
    """
    生成图片
    
    Args:
        prompts: {"cover": "封面描述", "img1": "配图1描述", ...}
        output_dir: 输出目录
    
    Returns:
        {"cover": "/path/to/cover.png", "img1": "/path/to/img1.png", ...}
    """
    output_dir.mkdir(parents=True, exist_ok=True)
    results = {}
    
    for key, prompt in prompts.items():
        output_path = output_dir / f"{key}.png"
        aspect = "2.35:1" if key == "cover" else "4:3"
        
        print(f"\n生成 {key} ({aspect})...")
        result = subprocess.run([
            PYTHON_BIN, str(GENERATE_IMAGE_SCRIPT),
            "--prompt", prompt,
            "--output", str(output_path),
            "--aspect-ratio", aspect
        ], capture_output=True, text=True)
        
        if result.returncode != 0:
            print(f"  错误: {result.stderr}")
            sys.exit(1)
        
        results[key] = str(output_path)
        print(f"  完成: {output_path}")
    
    return results

# ============================================================================
# Markdown → HTML
# ============================================================================

def md2html(md_content: str, style_name: str, img_urls: dict) -> str:
    """
    将 Markdown 转换为微信公众号 HTML
    
    Args:
        md_content: Markdown 内容
        style_name: 风格名称
        img_urls: {"img1": "https://...", "img2": "https://...", ...}
    
    Returns:
        HTML 字符串
    """
    style = STYLES.get(style_name, STYLES["purple"])

    # 清理：去掉 * 和各种引号（在解析 Markdown 之前，避免影响 HTML 结构）
    md_content = md_content.replace('*', '')  # 去掉所有 *
    md_content = md_content.replace('\u201c', '').replace('\u201d', '')  # 去掉中文双引号 ""
    md_content = md_content.replace('"', '')  # 去掉英文双引号

    # 替换图片占位符
    for key, url in img_urls.items():
        pattern = rf'<!--\s*IMAGE_\d+:.*?-->'
        if re.search(pattern, md_content):
            img_html = f'<p style="text-align:center;margin:20px 0"><img src="{url}" style="max-width:100%;border-radius:8px"></p>'
            md_content = re.sub(pattern, img_html, md_content, count=1)
    
    lines = md_content.split('\n')
    html_parts = []
    in_code_block = False
    code_content = []
    in_table = False
    table_rows = []
    
    for line in lines:
        # 代码块
        if line.startswith('```'):
            if in_code_block:
                code_html = '<br>'.join(code_content)
                html_parts.append(
                    f'<pre style="background:{style["code_bg"]};padding:16px;border-radius:8px;'
                    f'overflow-x:auto;font-size:14px;line-height:1.6;color:{style["code_color"]};'
                    f'margin:16px 0;white-space:pre-wrap"><code>{code_html}</code></pre>'
                )
                code_content = []
                in_code_block = False
            else:
                in_code_block = True
            continue
        
        if in_code_block:
            code_content.append(line.replace('<', '&lt;').replace('>', '&gt;'))
            continue
        
        # 表格
        if line.startswith('|'):
            if not in_table:
                in_table = True
                table_rows = []
            if '---' not in line:
                cells = [c.strip() for c in line.split('|')[1:-1]]
                table_rows.append(cells)
            continue
        elif in_table:
            table_html = '<table style="width:100%;border-collapse:collapse;margin:16px 0;font-size:14px">'
            for i, row in enumerate(table_rows):
                if i == 0:
                    table_html += '<tr>' + ''.join(
                        f'<th style="background:{style["quote_bg"]};padding:12px;border:1px solid #ddd;'
                        f'text-align:left;font-weight:600">{c}</th>' for c in row
                    ) + '</tr>'
                else:
                    table_html += '<tr>' + ''.join(
                        f'<td style="padding:12px;border:1px solid #ddd">{c}</td>' for c in row
                    ) + '</tr>'
            table_html += '</table>'
            html_parts.append(table_html)
            in_table = False
        
        if not line.strip():
            continue
        
        # 标题
        if line.startswith('# '):
            title = line[2:]
            html_parts.append(
                f'<h1 style="color:{style["title_color"]};font-size:24px;font-weight:700;'
                f'margin:24px 0 16px;line-height:1.4">{title}</h1>'
            )
        elif line.startswith('## '):
            title = line[3:]
            html_parts.append(
                f'<h2 style="color:{style["title_color"]};font-size:20px;font-weight:600;'
                f'margin:24px 0 12px;border-left:4px solid {style["title_color"]};'
                f'padding-left:12px;line-height:1.4">{title}</h2>'
            )
        elif line.startswith('### '):
            title = line[4:]
            html_parts.append(
                f'<h3 style="color:{style["title_color"]};font-size:17px;font-weight:600;'
                f'margin:20px 0 10px">{title}</h3>'
            )
        # 引用
        elif line.startswith('>'):
            quote = line[1:].strip()
            html_parts.append(
                f'<blockquote style="background:{style["quote_bg"]};'
                f'border-left:4px solid {style["title_color"]};padding:15px 20px;'
                f'margin:16px 0;color:#666;font-style:italic">{quote}</blockquote>'
            )
        # 列表
        elif line.startswith('- '):
            item = line[2:]
            item = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', item)
            html_parts.append(
                f'<p style="margin:8px 0;padding-left:20px;position:relative;'
                f'color:{style["text_color"]}"><span style="position:absolute;left:0;'
                f'color:{style["title_color"]}">•</span>{item}</p>'
            )
        elif re.match(r'^\d+\. ', line):
            item = re.sub(r'^\d+\. ', '', line)
            item = re.sub(r'\*\*(.+?)\*\*', r'<strong>\1</strong>', item)
            html_parts.append(
                f'<p style="margin:8px 0;padding-left:24px;color:{style["text_color"]}">{item}</p>'
            )
        # 分隔线
        elif line.strip() == '---':
            html_parts.append('<hr style="border:none;border-top:1px solid #eee;margin:24px 0">')
        # 斜体段落
        elif line.startswith('*') and line.endswith('*'):
            text = line[1:-1]
            html_parts.append(
                f'<p style="color:#888;font-size:14px;font-style:italic;'
                f'margin:16px 0;text-align:center">{text}</p>'
            )
        # 普通段落
        else:
            para = line
            para = re.sub(r'\*\*(.+?)\*\*', r'<strong style="color:#333">\1</strong>', para)
            para = re.sub(
                r'`([^`]+)`',
                rf'<code style="background:{style["code_bg"]};padding:2px 6px;'
                rf'border-radius:4px;font-size:14px;color:{style["code_color"]}">\1</code>',
                para
            )
            html_parts.append(
                f'<p style="margin:16px 0;line-height:1.8;color:{style["text_color"]}">{para}</p>'
            )

    return f'''<section style="font-family:{style["font"]};line-height:1.8;color:{style["text_color"]};padding:20px;max-width:100%">
{chr(10).join(html_parts)}
</section>'''

# ============================================================================
# 微信发布
# ============================================================================

def publish(html_content: str, title: str, digest: str, cover_path: str, img_paths: list) -> dict:
    """
    上传图片并发布到微信公众号草稿箱
    
    Returns:
        {"draft_id": "xxx", "thumb_media_id": "xxx", "img_urls": [...]}
    """
    import requests
    
    appid = os.environ["WECHAT_APPID"]
    secret = os.environ["WECHAT_SECRET"]
    
    # 获取 token
    print("\n获取 access_token...")
    token_resp = requests.get(
        "https://api.weixin.qq.com/cgi-bin/token",
        params={"grant_type": "client_credential", "appid": appid, "secret": secret}
    )
    token_data = token_resp.json()
    if "access_token" not in token_data:
        print(f"获取 token 失败: {token_data}")
        sys.exit(1)
    token = token_data["access_token"]
    print("  Token: OK")
    
    # 上传封面图
    print("\n上传封面图...")
    with open(cover_path, "rb") as f:
        thumb_resp = requests.post(
            f"https://api.weixin.qq.com/cgi-bin/material/add_material?access_token={token}&type=image",
            files={"media": ("cover.png", f, "image/png")}
        )
    thumb_data = thumb_resp.json()
    if "media_id" not in thumb_data:
        print(f"上传封面失败: {thumb_data}")
        sys.exit(1)
    thumb_media_id = thumb_data["media_id"]
    print(f"  media_id: {thumb_media_id}")
    
    # 上传配图
    print("\n上传配图...")
    img_urls = []
    for i, path in enumerate(img_paths, 1):
        with open(path, "rb") as f:
            resp = requests.post(
                f"https://api.weixin.qq.com/cgi-bin/media/uploadimg?access_token={token}",
                files={"media": (f"img{i}.png", f, "image/png")}
            )
        data = resp.json()
        if "url" not in data:
            print(f"上传配图{i}失败: {data}")
            sys.exit(1)
        img_urls.append(data["url"])
        print(f"  配图{i}: OK")
    
    # 替换 HTML 中的图片占位符
    for i, url in enumerate(img_urls):
        placeholder = f'<p style="text-align:center;margin:20px 0"><img src="" style="max-width:100%;border-radius:8px"></p>'
        img_html = f'<p style="text-align:center;margin:20px 0"><img src="{url}" style="max-width:100%;border-radius:8px"></p>'
        # 简单替换第一个空 src
        html_content = html_content.replace('src=""', f'src="{url}"', 1)
    
    # 创建草稿
    print("\n创建草稿...")
    draft_data = {
        "articles": [{
            "title": title[:64],  # 微信限制 64 字符
            "digest": digest[:120],
            "content": html_content,
            "thumb_media_id": thumb_media_id,
            "need_open_comment": 1,
            "only_fans_can_comment": 0
        }]
    }
    
    response = requests.post(
        f"https://api.weixin.qq.com/cgi-bin/draft/add?access_token={token}",
        data=json.dumps(draft_data, ensure_ascii=False).encode('utf-8'),
        headers={"Content-Type": "application/json; charset=utf-8"}
    )
    result = response.json()
    
    if "media_id" not in result:
        print(f"创建草稿失败: {result}")
        sys.exit(1)
    
    return {
        "draft_id": result["media_id"],
        "thumb_media_id": thumb_media_id,
        "img_urls": img_urls
    }

# ============================================================================
# 主函数
# ============================================================================

def main():
    parser = argparse.ArgumentParser(description="微信公众号文章发布工具")
    parser.add_argument("--markdown", "-m", required=True, help="Markdown 文件路径")
    parser.add_argument("--style", "-s", default="purple", choices=list(STYLES.keys()), help="CSS 风格")
    parser.add_argument("--images", "-i", type=int, default=3, choices=[0, 1, 2, 3], help="配图数量")
    parser.add_argument("--title", "-t", help="文章标题（默认从 Markdown 提取）")
    parser.add_argument("--digest", "-d", help="文章摘要（默认从 Markdown 提取）")
    parser.add_argument("--output-dir", "-o", default="/tmp/wechat_publish", help="输出目录")
    parser.add_argument("--skip-preflight", action="store_true", help="跳过预检")
    parser.add_argument("--skip-images", action="store_true", help="跳过图片生成（使用已有图片）")
    
    args = parser.parse_args()
    
    # 预检
    if not args.skip_preflight:
        if not preflight():
            sys.exit(1)
    
    # 读取 Markdown
    md_path = Path(args.markdown)
    if not md_path.exists():
        print(f"文件不存在: {md_path}")
        sys.exit(1)
    
    md_content = md_path.read_text(encoding="utf-8")
    
    # 提取标题和摘要
    title = args.title
    if not title:
        match = re.search(r'^#\s+(.+)$', md_content, re.MULTILINE)
        title = match.group(1) if match else "无标题"
    
    digest = args.digest
    if not digest:
        # 取第一段非标题文本
        lines = [l for l in md_content.split('\n') if l.strip() and not l.startswith('#')]
        digest = lines[0][:120] if lines else ""
    
    print(f"\n标题: {title}")
    print(f"摘要: {digest[:50]}...")
    print(f"风格: {args.style}")
    print(f"配图: {args.images} 张")
    
    output_dir = Path(args.output_dir)
    
    # 生成图片
    if not args.skip_images and args.images > 0:
        # 从 Markdown 提取图片描述
        img_comments = re.findall(r'<!--\s*IMAGE_(\d+):\s*(.+?)\s*-->', md_content)
        prompts = {"cover": f"{title}, tech illustration, purple theme"}
        for num, desc in img_comments[:args.images]:
            prompts[f"img{num}"] = desc
        
        print("\n" + "=" * 60)
        print("生成图片")
        print("=" * 60)
        img_paths = generate_images(prompts, output_dir)
    else:
        # 使用已有图片
        img_paths = {
            "cover": str(output_dir / "cover.png"),
            **{f"img{i}": str(output_dir / f"img{i}.png") for i in range(1, args.images + 1)}
        }
    
    # 转换 HTML（先用空 URL，后面替换）
    print("\n" + "=" * 60)
    print("转换 HTML")
    print("=" * 60)
    # 临时用空 URL
    temp_urls = {f"img{i}": "" for i in range(1, args.images + 1)}
    html_content = md2html(md_content, args.style, temp_urls)
    
    # 保存 HTML
    output_dir.mkdir(parents=True, exist_ok=True)
    html_path = output_dir / "article.html"
    html_path.write_text(html_content, encoding="utf-8")
    print(f"HTML 已保存: {html_path}")
    
    # 发布
    print("\n" + "=" * 60)
    print("发布到微信公众号")
    print("=" * 60)
    
    cover_path = img_paths["cover"]
    content_img_paths = [img_paths[f"img{i}"] for i in range(1, args.images + 1) if f"img{i}" in img_paths]
    
    result = publish(html_content, title, digest, cover_path, content_img_paths)
    
    # 报告
    print("\n" + "=" * 60)
    print("发布完成")
    print("=" * 60)
    print(f"""
文章信息
  标题: {title}
  摘要: {digest[:50]}...
  风格: {args.style}

图片
  封面 media_id: {result['thumb_media_id']}
  配图 URLs: {len(result['img_urls'])} 张

公众号
  草稿 ID: {result['draft_id']}
  请前往公众号后台查看并发布
""")


if __name__ == "__main__":
    main()
