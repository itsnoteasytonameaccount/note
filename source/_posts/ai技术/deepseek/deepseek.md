---
title: 接入deepseek
date: 2026/08/23
tags:
- ai
- deepseek
---

# codex接入deepseek

参考[接入 Codex | DeepSeek API Docs](https://api-docs.deepseek.com/zh-cn/quick_start/agent_integrations/codex)

创建`~/.codex/models.json`

声明元数据，具体内容上面的地址有提供

再修改`~/.codex/config.toml`指定模型提供方

```toml
model = "deepseek-v4-flash"
model_provider = "deepseek"
preferred_auth_method = "apikey"
forced_login_method = "api"
model_reasoning_effort = "high"
model_catalog_json = "~/.codex/models.json"

[model_providers.deepseek]
name = "deepseek"
base_url = "https://api.deepseek.com/"
wire_api = "responses"
experimental_bearer_token = "<你的 DeepSeek API Key>"
```



# cursor接入deepseek

官网说deepseek使用和openAI兼容的API格式

> DeepSeek API 使用与 OpenAI/Anthropic 兼容的 API 格式，通过修改配置，您可以使用 OpenAI/Anthropic SDK 来访问 DeepSeek API，或使用与 OpenAI/Anthropic API 兼容的软件。

但是配了没用，还是报错，只能曲线救国了，在cursor里安装codex插件，然后使用codex插件配置好的deepseek模型。

不能用的原因通过codex分析了一下

> **结论**
> DeepSeek 的配置其实已经写进 Cursor（开关、Base URL、模型名都在），但请求**根本没发到 DeepSeek**——被 Cursor 自己的后端拦截了。Cursor 当前是免费账号（`membershipType = free`），免费版不允许使用指定名称模型（named model），后端返回错误码 51，所以表现成“接入了但没起作用”。