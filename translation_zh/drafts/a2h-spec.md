# A2H - 代理到人类协议


## 概述

A2H 是一项允许代理请求人类交互的服务


## 为什么需要另一个协议？

MCP 和 A2A 还不够

## 应该

- 如果设置了 A2H_BASE_URL 和 A2H_API_KEY 环境变量，客户端应该尊重这些变量，以允许基于 OAuth2 的简单身份验证访问 REST 服务。

## 核心协议

### 作用域

A2H 协议支持两个作用域：

- 代理端：由代理消费的 API，用于请求人类交互
- （可选）管理端：由管理员或 Web 应用程序消费的 API，用于管理人类及其联系渠道

这种分离允许代理查询和查找要联系的人类，而无需向代理暴露人类的联系详情。A2H 提供商的责任是通过该人类首选的联系渠道将代理请求转发给相应的人类。

### 对象

```
apiVersion: proto.a2h.dev/v1alpha1
kind: Message
metatdata:
  uid: "123"
spec: # spec sent by agent
  message: "" # message from the agent
  response_schema:
   # optional, json schema for the response,
  channel_id: 
status: # status resolved by a2h server
  humanMessage: "" # message from the human
  response:
    # optional, matches spec schema
```

```
apiVersion: proto.a2h.dev/v1alpha1
kind: NewConversation
metadata:
  uid: "abc"
spec: # spec sent by a2h server
  message: "" # message from the agent
  channel_id: "123" # channel id to use for future conversations
  response_schema:
   # optional, json schema for the response,
```



#### HumanContact

```json
{
  "run_id": "run_123",
  "call_id": "call_456",
  "spec": {
    "msg": "I've tried using the tool to refund the customer but its returning a 500 error. Can you help?",
    "channel": {
      "slack": {
        "channel_or_user_id": "U1234567890",
        "context_about_channel_or_user": "Support team lead"
      }
    },
  },
}
```

HumanContact 表示一个人类交互请求。它包含：

- `run_id` (string)：运行的唯一标识符
- `call_id` (string)：联系请求的唯一标识符
- `spec` (HumanContactSpec)：联系请求的规格说明
- `status` (HumanContactStatus, optional)：联系请求的当前状态

HumanContactSpec 包含：
- `msg` (string)：发送给人类的消息
- `subject` (string, optional)：联系请求的主题
- `channel` (ContactChannel, optional)：用于联系的渠道
- `response_options` (ResponseOption[], optional)：可用的响应选项
- `state` (object, optional)：附加状态信息

HumanContactStatus 包含：
- `requested_at` (datetime, optional)：请求联系的时间
- `responded_at` (datetime, optional)：人类响应的时间
- `response` (string, optional)：人类的响应
- `response_option_name` (string, optional)：所选响应选项的名称
- `slack_message_ts` (string, optional)：Slack 消息时间戳（如适用）
- `failed_validation_details` (object, optional)：验证失败的详情

#### FunctionCall

示例：
```json
{
  "run_id": "run_789",
  "call_id": "call_101",
  "spec": {
    "fn": "process_payment",
    "kwargs": {
      "amount": 100.00,
      "currency": "USD",
      "recipient": "merchant_123"
    },
    "channel": {
      "email": {
        "address": "ap@example.com",
      }
    },
  },
  "status": {
    "requested_at": "2024-03-20T11:00:00Z",
    "responded_at": "2024-03-20T11:02:00Z",
    "approved": true,
    "comment": "Payment looks good, approved",
    "user_info": {
      "name": "John Doe",
      "role": "Finance Manager"
    },
    "slack_message_ts": "1234567890.123457"
  }
}
```

FunctionCall 表示一个人类审批函数执行的请求。它包含：

- `run_id` (string)：运行的唯一标识符
- `call_id` (string)：函数调用的唯一标识符
- `spec` (FunctionCallSpec)：函数调用的规格说明
- `status` (FunctionCallStatus, optional)：函数调用的当前状态

FunctionCallSpec 包含：
- `fn` (string)：要调用的函数
- `kwargs` (object)：函数的关键字参数
- `channel` (ContactChannel, optional)：用于联系的渠道
- `reject_options` (ResponseOption[], optional)：可用的拒绝选项
- `state` (object, optional)：附加状态信息

FunctionCallStatus 包含：
- `requested_at` (datetime, optional)：请求审批的时间
- `responded_at` (datetime, optional)：人类响应的时间
- `approved` (boolean, optional)：函数调用是否被批准
- `comment` (string, optional)：人类的任何评论
- `user_info` (object, optional)：响应用户的信息
- `slack_context` (object, optional)：Slack 特定上下文
- `reject_option_name` (string, optional)：所选拒绝选项的名称
- `slack_message_ts` (string, optional)：Slack 消息时间戳（如适用）
- `failed_validation_details` (object, optional)：验证失败的详情

#### ContactChannel

示例：
```json
{
  "slack": {
    "channel_or_user_id": "U1234567890",
    "context_about_channel_or_user": "Support team lead",
    "allowed_responder_ids": ["U1234567890", "U2345678901"],
    "experimental_slack_blocks": true,
    "thread_ts": "1234567890.123456"
  }
}
```

或

```json
{
    "email": {
        "address": "ap@example.com",
        "context_about_user": "Accounts Payable",
        "in_reply_to_message_id": "1234567890",
        "references_message_id": "1234567890",
        "template": "<html><body>...</body></html>"
    }
}
```

ContactChannel 表示可以联系人类的渠道。该协议支持多种渠道类型：

1. SlackContactChannel：
   - `channel_or_user_id` (string)：Slack 频道或用户 ID
   - `context_about_channel_or_user` (string, optional)：附加上下文
   - `bot_token` (string, optional)：用于身份验证的 Bot token
   - `allowed_responder_ids` (string[], optional)：允许响应者的 ID
   - `experimental_slack_blocks` (boolean, optional)：启用实验性 blocks
   - `thread_ts` (string, optional)：线程消息的时间戳

2. SMSContactChannel：
   - `phone_number` (string)：要联系的电话号码
   - `context_about_user` (string, optional)：关于用户的附加上下文

3. WhatsAppContactChannel：
   - `phone_number` (string)：要联系的电话号码
   - `context_about_user` (string, optional)：关于用户的附加上下文

#### Human（代理端）

从代理的角度来看，人类是一个具有名称和描述的对象。

#### Human（管理端）

从管理员的角度来看，人类是一个具有名称、描述和按优先级排序的联系渠道列表的对象，包含详细信息。

### 代理端点


#### POST /human_contacts

#### GET /human_contacts/:call_id

#### POST /function_calls

#### GET /function_calls/:call_id


## 扩展协议

- 管理端人类
- 代理端人类获取
- 代理端人类搜索
- 代理端渠道列表
- 代理端渠道验证

### 对象

#### Human（代理端）

从代理的角度来看，人类是一个具有名称和描述的对象。

#### Human（管理端）

从管理员的角度来看，人类是一个具有名称、描述和按优先级排序的联系渠道列表的对象，包含详细信息。

### 代理端点

#### GET /channels

返回可用的联系渠道及其支持的字段

示例响应：

```json
{
    "channels": {
        "slack": {
            "channelOrUserId": {
                "type": "string",
                "description": "The Slack channel or user ID to send messages to"
            },
            "contextAboutChannelOrUser": {
                "type": "string", 
                "description": "Additional context about the Slack channel or user"
            }
        },
        "email": {
            "address": {
                "type": "string",
                "description": "Email address to send messages to"
            },
            "contextAboutUser": {
                "type": "string",
                "description": "Additional context about the email recipient"
            },
            "inReplyToMessageId": {
                "type": "string",
                "description": "The message ID of the email to reply to"
            },
            "referencesMessageId": {
                "type": "string",
                "description": "The message ID of the email to reference"
            }
        }
    }
}
```

#### GET /humans

返回可供交互的人类列表

示例响应：

```json
{
    "humans": [
        {
            "id": "654",
            "name": "Jane Doe",
            "description": "Jane Doe is a human who knows about technology and entrepreneurship",
        },
        {
            "id": "123",
            "name": "John Doe",
            "description": "John Doe is a human who knows about sales and marketing"
        }
    ]
}
```

#### GET /humans/search?q=

按名称或描述搜索人类

示例响应：

```json
{
    "humans": [
        {
            "id": "654",
            "name": "Jane Doe",
            "description": "Jane Doe is a human who knows about technology and entrepreneurship",
        },
    ]
}
```

### 管理端点


#### POST /humans

注册一个新的人类以供代理联系

示例请求：

```json
{
    "name": "John Doe",
    "description": "John Doe is a human who knows about sales and marketing",
    "prioritizedContactChannels": [
        {
            "slack": {
                "channelOrUserId": "U1234567890",
            }
        },
        {
            "email": {
                "address": "john.doe@example.com",
            }
        }
    ]
}
```



#### GET /humans/:id

按 ID 获取人类

示例响应：

```json
```
