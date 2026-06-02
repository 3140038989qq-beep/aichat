# 图片/表情包处理链路

> 最后更新: 2026-06-02

## 问题与修复历程

### 问题1: 表情包崩溃
- **症状**: 企业微信发 `emotion` 类型的表情包，wechatpy 映射为 `UnknownMessage` → `NotImplementedError`
- **根因**: `wechatcomapp_message.py` 的 `__init__` 只处理了 `text/voice/image`，没有 `emotion` 分支

### 问题2: 图片零回复
- **症状**: 发图片没有任何 AI 回复，只下载存到本地但不处理
- **根因**: `chat_channel.py` 里 IMAGE 类型的处理只做了 `memory.USER_IMAGE_CACHE[]` 缓存，没有调用视觉识别

### 问题3: 视觉识别 404
- **症状**: 所有图片都被识别成"加班"
- **根因**: `DoubaoBot.__init__()` 直接用全局 `model`（deepseek-chat）当豆包模型名，豆包 API 不认识返回 404
- **日志证据**: `Vision API error: HTTP 404: model deepseek-chat does not exist`

### 问题4: 响应慢
- **根因**: 原图直传 base64（可能 3MB+），用 pro 大杯模型
- **优化**: PIL 压缩到 512px JPEG quality 75 → 豆包 lite 模型 → prompt 精简

## 修改的文件

### 1. `channel/wechatcom/wechatcomapp_message.py`
新增 `emotion` 类型处理：
```python
elif msg.type == "emotion":
    self.ctype = ContextType.IMAGE
    self.content = TmpDir().path() + msg.id + ".png"
    media_id = msg._data.get("MediaId", msg.id) if hasattr(msg, "_data") else msg.id
    # download_emotion() 通过 client.media.download(media_id) 下载
```

### 2. `channel/chat_channel.py`
新增 `_handle_image()` 方法：
```python
def _handle_image(self, context) -> Reply:
    # 1. PIL 压缩图片: 最大512px, JPEG quality 75
    # 2. 豆包 lite 模型 call_vision(img_url, prompt, model="doubao-seed-2-0-lite-260215")
    # 3. 识别结果转文本喂给对话Bot: "博士给你发了一张图: {vision_result}。回应他"
```

Vision Prompt:
```
描述这张图的内容，并推测发送者想表达什么
（分享日常/炫耀/撒娇/吐槽/搞笑/累/饿/想你了/分享风景/自拍等）
输出格式:【画面:xxx】【意图:xxx】 限60字
```

### 3. `models/doubao/doubao_bot.py`
修复模型名检测：
```python
# 之前: 直接用 conf().get("model") 不管是不是豆包模型
# 现在: 检查模型名是否以 "doubao" 开头，否则默认用 doubao-seed-2-0-pro-260215
if cfg_model.startswith("doubao"):
    model = cfg_model
else:
    model = "doubao-seed-2-0-pro-260215"
```

## 数据流

```
图片/表情包
  → wechatcomapp_message.py: 解析类型, 下载媒体文件
  → chat_channel.py: _handle_image()
    → PIL 压缩 (512px JPEG)
    → base64 编码
    → 豆包 lite call_vision()
    → 【画面: xxx】【意图: xxx】
    → "博士给你发了一张图: {结果}"
    → DeepSeek + W 角色Prompt → 角色回复
```
