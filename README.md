<p align="center"><img src= "https://github.com/user-attachments/assets/eca9a9ec-8534-4615-9e0f-96c5ac1d10a3" alt="Long" width="550" /></p>

<p align="center">
  <strong>维什戴尔 · 企业微信 AI 机器人</strong><br/>
  🐉 long 的个人定制版
</p>

---

## 📋 简介

这是一个预配置好的 **维什戴尔/W（明日方舟）角色扮演 AI 机器人**，通过企业微信自建应用接入。

**本仓库包含：**
- ✅ 维什戴尔完整人设（嘴硬心软、短文本轰炸）
- ✅ DeepSeek 模型配置模板（需自己填入 API Key）
- ✅ 企业微信应用通道配置模板
- ✅ `config.json` 已配置好除密钥外的所有参数

## ⚠️ 使用前必读

> **本仓库不含任何 API Key**，`config.json` 已被 `.gitignore` 忽略，不会上传到 GitHub。
>
> 要运行这个机器人，你需要：
> 1. 在 [DeepSeek 平台](https://platform.deepseek.com/api_keys) 申请自己的 API Key
> 2. 在企业微信后台创建自建应用，获取凭证
> 3. 编辑本地的 `config.json` 填入上述信息

## 🚀 快速启动

### 1. 安装依赖

```bash
pip3 install -r requirements.txt
pip3 install -r requirements-optional.txt
pip3 install -e .    # 安装 Cow CLI（可选但推荐）
```

### 2. 修改配置

编辑根目录下的 `config.json`，填入你自己的 API Key 和企业微信应用凭证：

```json
{
  "deepseek_api_key": "你的 DeepSeek API Key",
  "wechatcom_corp_id": "你的企业ID",
  "wechatcomapp_token": "你的Token",
  "wechatcomapp_aes_key": "你的AES Key",
  "wechatcomapp_agent_id": "你的应用ID",
  "wechatcomapp_secret": "你的应用Secret"
}
```

### 3. 运行

```bash
python3 app.py
```

或后台运行：
```bash
nohup python3 app.py > run.log 2>&1 &
```

### 4. 配置企业微信回调

需要将你的服务器地址配置到企业微信应用的回调 URL 中，详见 [企业微信官方文档](https://developer.work.weixin.qq.com/document/path/90238)。

## 📝 自定义角色

修改 `config.json` 中的 `character_desc` 字段即可替换为你的角色设定。格式参考：

```json
{
  "character_desc": "# 角色设定\n\n## 1. 核心人设\n..."
}
```

## 📄 许可证

本项目基于 [MIT 许可证](LICENSE)，原始项目由 [zhayujie](https://github.com/zhayujie) 开发。
