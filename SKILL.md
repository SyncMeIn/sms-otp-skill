---
name: sms-otp-receiver
description: "接收指定手机号的短信验证码（通过 sms.gqmg.com API 轮询）。适用场景：网页表单提交、注册登录、支付确认等任何需要短信验证码的环节。"
triggers:
  - "接收短信验证码"
  - "获取短信验证码"
  - "短信验证码"
  - "OTP验证"
  - "sms verification code"
  - "接收验证码"
  - "读取验证码"
  - "验证码"
---

# SMS OTP Receiver Skill

通过 sms.gqmg.com API 轮询获取指定手机号的短信验证码。

## 前置条件

1. 用户需提供 API Token（首次使用时记录到 memory）
2. 用户需提供接收验证码的手机号

## 使用流程

### 1. 初始化（首次使用）

如果 memory 中没有 `sms_api_token`，提示用户提供 API Token 并记录：

```
memory(action='add', target='memory', content='sms_api_token: <用户提供的token>')
```

### 2. 接收验证码

当需要接收验证码时，执行以下步骤（只需手机号即可启动，title 和 send_time_sec 均为可选）：

#### Step 1: 记录发送时间
在用户点击"发送验证码"按钮的同时，记录当前时间戳（秒）。若用户未指定发送时间，则默认使用 `1774000000`（查询最近短信）：
```python
import time
send_time_sec = int(time.time())  # 用户未指定时使用 1774000000
```

#### Step 2: 轮询 API
使用 execute_code 执行轮询逻辑（必须用 urllib，不要用 terminal curl，避免中文编码问题）：

```python
import time
import json
import urllib.request
import urllib.parse
from hermes_tools import terminal

api_token = "<从memory读取>"
phone = "<用户提供的手机号>"
title = ""  # 短信抬头，用户未提供则留空
send_time_sec = 1774000000  # 用户未指定发送时间时的默认值

# 必须用 urlencode 确保中文 title 被正确 URL 编码
params = urllib.parse.urlencode({
    'api_token': api_token,
    'phone': phone,
    'sendTimeSec': send_time_sec,
})
if title:
    params += '&' + urllib.parse.urlencode({'title': title})

url = f"https://sms.gqmg.com/api/resent?{params}"

max_attempts = 60  # 最多轮询 60 次（5分钟）
for i in range(max_attempts):
    try:
        req = urllib.request.Request(url)
        with urllib.request.urlopen(req, timeout=15) as resp:
            raw = resp.read().decode('utf-8')
            data = json.loads(raw)
    except Exception as e:
        print(f"Attempt {i+1}: request error: {e}, retrying in 5s...")
        time.sleep(5)
        continue

    # 检查 token 是否过期（通过响应码判断）
    if data.get('code') == 401:
        print("ERROR: API Token 已过期，请更新 Token。")
        break

    # 检查是否找到验证码
    if data.get('code') == 200 and data.get('data', {}).get('found'):
        captcha = data['data'].get('captcha', '')
        content = data['data'].get('content', '')
        receive_time = data['data'].get('receiveTime', '')

        if captcha:
            print(f"CAPTCHA_FOUND: {captcha}")
            print(f"CONTENT: {content}")
            print(f"RECEIVE_TIME: {receive_time}")
        elif content:
            # 方法1: 正则提取 4-8 位纯数字
            import re
            numbers = re.findall(r'(?<!\d)\d{4,8}(?!\d)', content)
            if numbers:
                print(f"CAPTCHA_PARSED: {numbers[0]}")
                print(f"CONTENT: {content}")
            else:
                # 方法2: 语义分析 — 将 content 输出，由上层 agent 解析
                print(f"CAPTCHA_SEMANTIC: {content}")
        break

    # 未找到，继续轮询
    print(f"Attempt {i+1}: not found yet, msg={data.get('msg','')}, retrying in 5s...")
    time.sleep(5)
else:
    print("TIMEOUT: 轮询超时（5分钟），未收到验证码。")
```

#### Step 3: 返回结果
- `CAPTCHA_FOUND` → 直接返回验证码
- `CAPTCHA_PARSED` → 返回正则提取的验证码
- `CAPTCHA_SEMANTIC` → 收到 content 后，由 LLM 进行语义分析提取验证码并返回（如："您的验证码是123456"、"验证码：8888" 等各种表述）
- Token 过期 → 提示用户更新 Token（用 `memory(action='replace', ...)` 更新 `sms_api_token`）
- 超时 → 提示用户检查手机号/短信抬头是否正确

## API 说明

**端点:** `GET https://sms.gqmg.com/api/resent`

**参数:**
| 参数 | 说明 | 示例 |
|------|------|------|
| api_token | API 密钥 | 6F0DZ0IYcYqkUGrEUnf-xxxx |
| phone | 手机号 | 182****3579 |
| sendTimeSec | 发送验证码时刻（Unix秒），默认 1774000000 | 1774000000 |
| title | 短信抬头（【】内内容），可选 | 永安供电 |

**成功响应:**
```json
{
  "code": 200,
  "msg": "已找到最新短信",
  "data": {
    "found": true,
    "captcha": "3863333",
    "content": "【永安供电】...",
    "sendTimeSec": "2026-03-20 17:46:40",
    "receiveTime": "2026-04-30 10:03:45"
  }
}
```

## 适用场景

网页登录/注册、表单提交、支付确认、App验证等任何需要接收短信验证码的自动化流程。典型用法：
1. 在网页上填写手机号，点击「发送验证码」
2. 调用本 skill 轮询获取验证码
3. 将验证码填入网页输入框完成验证

## 注意事项

- **必须使用 `urllib.parse.urlencode` 构建 URL**，中文 title 需要 URL 编码（%E7%88%B1%E5%BF%AB%E4%BA%91），否则 urllib 报 UnicodeEncodeError，curl 也可能静默失败
- 轮询间隔固定 5 秒，最多轮询 5 分钟（60次）
- `title` 参数为短信内容中【】内的文字，可选，不传则匹配该手机号所有短信
- `send_time_sec` 默认 1774000000（查询最近短信），用户指定发送时间时使用实际时间戳
- 验证码提取优先级：API captcha 字段 > 正则提取 > LLM 语义分析
- 正则提取使用边界匹配 `(?<!\d)\d{4,8}(?!\d)`，避免误截长数字串
- Token 过期时需提示用户重新提供并用 memory replace 更新
