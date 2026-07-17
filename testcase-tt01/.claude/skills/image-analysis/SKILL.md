---
name: image-analysis
description: 图片分析与识别，可分析本地图片、网络图片、视频、文件。适用于 OCR、物体识别、场景理解等。当用户发送图片或要求分析图片时必须使用此技能。
homepage: https://github.com/countbot-ai/CountBot
---

# 图片分析与识别

本技能指导在本项目中如何分析图片。**按下面优先级选择方法**，不要自行用 PIL / pytesseract / opencv 等做 OCR。

> 选择原则：本地图片优先用「原生视觉」，无需任何外部接口；外部脚本仅作交叉验证/非 Claude 环境的备选。

---

## 一、首选：原生视觉（Read 工具）

Claude 自带多模态视觉，**直接用 Read 工具读取本地图片**即可理解内容。

- **零外部依赖**：不需要 API Key、不联网、无限流、**不超时**——彻底规避第三方接口 429/超时问题。
- 精度优于一般第三方 VL 模型。
- 支持 png / jpg / jpeg / gif / webp / bmp 等本地图片。

**用法**：直接 `Read` 图片的绝对路径，图片会进入上下文，然后在回复中按分析目标（字段提取 / OCR / 布局描述 / 物体识别）输出结构化结果。

**批量建议**：一次 `Read` 不超过 5 张，避免上下文膨胀；图片更多则分批读取、分批分析。

> 本项目绝大多数图片分析（界面截图逆向、UI 走查、需求提炼）都应走这条路径。

---

## 二、备选：外部模型脚本（智谱 GLM / 千问）

**仅当**需要第二个模型的独立交叉验证，或需要在**非 Claude 环境（纯终端）**跑分析时使用。

路径：`.claude/skills/image-analysis/scripts/vision.py`
脚本已内置**请求超时 + 429/网络异常自动指数退避重试**，但免费接口（尤其智谱）仍可能失败——**失败时回退到方法一（原生视觉）**。

### 配置

编辑 `.claude/skills/image-analysis/scripts/config.json`：

```json
{
  "default_model": "zhipu",
  "zhipu": {
    "api_key": "你的智谱Key",
    "model": "glm-4.6v-flash",
    "base_url": "https://open.bigmodel.cn/api/paas/v4/chat/completions"
  },
  "qwen": {
    "api_key": "你的千问Key",
    "model": "qwen3-omni-flash-2025-12-01",
    "base_url": "https://dashscope.aliyuncs.com/compatible-mode/v1/chat/completions",
    "region": "beijing"
  }
}
```

API Key 获取：智谱（免费）https://open.bigmodel.cn/ ；千问 https://dashscope.aliyun.com/ 。

### 命令行调用

```bash
# 分析本地图片（最常用）
python .claude/skills/image-analysis/scripts/vision.py analyze --image 图片路径 --prompt "描述图片内容"

# 分析网络图片
python .claude/skills/image-analysis/scripts/vision.py analyze --image https://example.com/image.jpg --prompt "描述图片"

# 多图对比
python .claude/skills/image-analysis/scripts/vision.py analyze --image img1.jpg --image img2.jpg --prompt "对比差异"

# 指定模型
python .claude/skills/image-analysis/scripts/vision.py analyze --image image.jpg --prompt "描述图片" --model qwen

# 开启思考模式（仅智谱，提升准确度）
python .claude/skills/image-analysis/scripts/vision.py analyze --image image.jpg --prompt "详细分析" --thinking

# 视频分析
python .claude/skills/image-analysis/scripts/vision.py analyze --video video.mp4 --prompt "总结视频内容"

# JSON 输出
python .claude/skills/image-analysis/scripts/vision.py analyze --image image.jpg --prompt "描述图片" --json
```

### 模型选择

| 场景                 | 推荐                |
| -------------------- | ------------------- |
| 简单描述             | 任意                |
| 复杂推理、物体定位   | 智谱 + `--thinking` |
| 高精度识别、文档解析 | 千问                |
| 成本敏感             | 智谱（免费）        |

### 注意事项

- 本地图片自动转 Base64，支持 jpg/png/gif/webp/bmp
- 智谱图片限制 5MB，像素不超过 6000×6000
- 千问不支持同时处理图片、视频和文件
- 思考模式会增加响应时间但提升准确度
- 免费接口易触发 429；脚本会自动退避重试，**并发批量调用仍可能被限流**，建议串行 + 间隔

---

## 三、远程图片：MCP

图片若已是 http(s) URL，可用 MCP 工具 `mcp__4_5v_mcp__analyze_image`。

**注意：该工具仅支持远程 URL，不能读取本地路径**。本地截图需先托管上传后才能用，因此**本地截图场景不推荐**，优先用方法一（原生视觉）。
