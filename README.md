# 🤖 闲鱼AI智能客服机器人 - XyAutoAgent

[![Python Version](https://img.shields.io/badge/python-3.8%2B-blue)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-MIT-green)]()
[![Architecture](https://img.shields.io/badge/architecture-Multi--Agent-brightgreen)]()

一个基于**多智能体AI系统**的闲鱼平台自动化客服解决方案，实现7×24小时智能值守、自动议价、实时对话和上下文感知应答。通过**AI Agent意图路由** + **专家系统** + **SQLite数据库上下文管理**，提供智能化的客户交互体验。

---

## 📋 项目概述

### 核心能力

XyAutoAgent 是一个**全自动化智能客服系统**，专为闲鱼二手交易平台设计。系统通过实时连接闲鱼IM（DingTalk WebSocket协议），自动扮演卖家角色，智能处理买家咨询：

| 功能模块 | 描述 | 技术实现 |
|---------|------|--------|
| **智能路由系统** | 自动识别用户意图，分发给相应专家Agent | LLM意图分类 + 规则引擎 |
| **议价专家** | 支持阶梯降价策略，动态调温控制 | 议价次数追踪 + 温度调整 |
| **技术支持专家** | 处理产品参数、规格咨询，网络搜索增强 | 技术关键词识别 + 正则模式匹配 |
| **客服专家** | 通用场景回复 | 默认对话Agent |
| **上下文感知对话** | 保留完整对话历史，支持多轮交互 | SQLite数据库 + ChatContextManager |
| **人工接管模式** | 支持实时切换AI/人工值守 | 关键词识别 + 会话状态管理 |

---

## 🏗️ 技术架构

### 系统整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         闲鱼平台 (Xianyu)                        │
│                   (WebSocket: wss-goofish.dingtalk.com)         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  WebSocket连接层 │
                    │  (main.py)      │
                    └────────┬────────┘
                             │
        ┌────────────┬────────┼────────┬────────────┐
        │            │        │        │            │
    ┌───▼────┐  ┌──▼───┐ ┌──▼───┐ ┌─▼────┐   ┌──▼────┐
    │消息解析 │  │Token │ │心跳  │ │断线  │   │Cookie │
    │(同步包) │  │管理  │ │管理  │ │重连  │   │签名   │
    └───┬────┘  └──┬───┘ └──┬───┘ └─┬────┘   └──┬────┘
        │          │        │       │           │
        └──────────┼────────┼───────┴───────────┘
                   │        │
                   ▼        ▼
            ┌──────────────────────┐
            │   AI Agent智能引擎   │
            │  (XianyuAgent.py)   │
            └──────────┬───────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        │    (意图路由) │              │
        │   IntentRouter             │
        │              │              │
        ▼              ▼              ▼
    ┌────────┐   ┌─────────┐   ┌──────────┐
    │ClassifyA│  │PriceAgent│  │TechAgent │
    │ (LLM)  │  │(议价)    │  │(技术)    │
    └────────┘   └─────────┘   └──────────┘
        │              │              │
        │              │              ▼
        │              │         ┌──────────┐
        │              │         │DefaultAgent
        │              │         │(通用)
        │              │         └──────────┘
        │              │              │
        └──────────────┼──────────────┘
                       │
                ┌──────▼──────┐
                │ 安全过滤器  │
                │ (检测违规词)│
                └──────┬──────┘
                       │
         ┌─────────────┼──────────────┐
         │             │              │
         ▼             ▼              ▼
    ┌────────┐   ┌─────────┐   ┌──────────┐
    │SQLite  │   │上下文   │   │消息发送  │
    │数据库  │   │管理器   │   │(IM)     │
    └────────┘   └─────────┘   └──────────┘
```

### AI Agent多智能体系统架构

#### 意图路由层（Intent Routing Layer）
```
用户消息输入
    ↓
├─ 第一级：关键词匹配 (技术类优先)
│   ├─ 关键词库：['参数', '规格', '型号', '连接', '对比']
│   └─ 若匹配 → 路由至 TechAgent
│
├─ 第二级：正则模式匹配
│   ├─ 技术模式：和.+比
│   └─ 价格模式：\d+元, 能少\d+
│
└─ 第三级：LLM智能分类 (Fallback)
    └─ 调用ClassifyAgent进行精准意图识别
```

#### 专家Agent系统（Expert Agent System）
```
BaseAgent (基类)
├── ClassifyAgent
│   ├─ 用途：意图分类（内部路由）
│   ├─ 输入：用户消息 + 商品信息 + 对话历史
│   └─ 输出：意图标签 {'price', 'tech', 'default', 'no_reply'}
│
├── PriceAgent (议价专家)
│   ├─ 用途：处理砍价/讨价还价请求
│   ├─ 动态温度：T = 0.3 + 议价轮次 * 0.05
│   │   (第0轮: 0.3, 第1轮: 0.35, 第5轮: 0.55)
│   ├─ 输入：{user_msg, item_desc, context, bargain_count}
│   └─ 输出：降价建议/拒绝文案
│
├── TechAgent (技术专家)
│   ├─ 用途：技术咨询、产品对比、参数解答
│   ├─ 输入：{user_msg, item_desc, context}
│   └─ 输出：技术答复
│
└── DefaultAgent (通用客服)
    ├─ 用途：其他场景通用回复
    ├─ 输入：{user_msg, item_desc, context}
    └─ 输出：客服标准回复
```

---

## 🛠️ 技术栈

### 后端技术
| 组件 | 技术 | 版本 | 用途 |
|------|------|------|------|
| **运行时** | Python | 3.10+ | 核心应用 |
| **AI/LLM** | OpenAI SDK | 1.65.5 | 调用大语言模型（通义千问等） |
| **即时通信** | websockets | 13.1 | WebSocket连接闲鱼IM系统 |
| **HTTP请求** | requests | 2.32.3 | API调用、Cookie管理、登录验证 |
| **日志系统** | loguru | 0.7.3 | 结构化日志、性能监控 |
| **环境配置** | python-dotenv | 1.0.1 | 环境变量管理 |

### 数据库技术
| 数据库 | 版本 | 用途 | 存储内容 |
|--------|------|------|--------|
| **SQLite** | 3.x | 轻量级持久化 | 对话历史、议价次数、商品信息 |

**数据库表设计：**

```sql
-- 消息表：存储用户与AI的完整对话历史
CREATE TABLE messages (
    id INTEGER PRIMARY KEY,              -- 消息ID
    user_id TEXT,                        -- 用户ID
    item_id TEXT,                        -- 商品ID
    chat_id TEXT,                        -- 会话ID
    role TEXT,                           -- 角色 {'user', 'assistant', 'system'}
    content TEXT,                        -- 消息内容
    timestamp DATETIME                   -- 时间戳
);
-- 索引：加速对话历史查询
CREATE INDEX idx_chat_id ON messages (chat_id);
CREATE INDEX idx_timestamp ON messages (timestamp);

-- 议价次数表：追踪每个会话的议价轮次
CREATE TABLE chat_bargain_counts (
    chat_id TEXT PRIMARY KEY,            -- 会话ID
    count INTEGER,                       -- 议价次数
    last_updated DATETIME                -- 最后更新时间
);

-- 商品信息表：缓存商品数据
CREATE TABLE items (
    item_id TEXT PRIMARY KEY,            -- 商品ID
    data TEXT,                           -- 商品完整信息(JSON)
    price REAL,                          -- 价格
    description TEXT,                    -- 商品描述
    last_updated DATETIME                -- 更新时间
);
```

### 容器化部署
| 工具 | 配置 | 用途 |
|------|------|------|
| **Docker** | Dockerfile (多阶段构建) | 镜像构建 |
| **Docker Compose** | docker-compose.yml | 服务编排 |

---

## 🏛️ 核心模块详解

### 1. WebSocket连接层 (`main.py` - XianyuLive 类)

**职责**：实时连接闲鱼IM系统，处理消息收发和连接管理

**核心功能**：
- WebSocket长连接维持（心跳、Token续约、断线重连）
- 消息同步包解析（MessagePack格式）
- 发送消息到闲鱼平台
- 人工/AI模式切换管理

**关键配置**：
```python
heartbeat_interval = 15       # 心跳间隔（秒）
token_refresh_interval = 3600 # Token刷新（秒）
manual_mode_timeout = 3600    # 人工模式超时（秒）
message_expire_time = 300000  # 消息过期时间（毫秒）
```

### 2. 闲鱼API层 (`XianyuApis.py`)

**职责**：与闲鱼后台交互，管理登录态和会话信息

**核心接口**：
```python
get_token(device_id)           # 获取实时Token
hasLogin()                     # 检查登录状态
get_item_info(item_id)         # 获取商品详情
clear_duplicate_cookies()      # 清理并更新Cookie
```

**安全机制**：
- MD5签名算法 (`generate_sign()`)
- Cookie持久化和动态更新
- 风控检测和自动恢复（RGV587）
- 登录状态验证

### 3. AI智能引擎 (`XianyuAgent.py`)

#### 3.1 意图路由器（IntentRouter）
```python
# 三级路由策略
def detect(user_msg, item_desc, context):
    Step 1: 技术类关键词检查（高优先级）
    Step 2: 技术类正则检查  
    Step 3: 价格类关键词检查
    Step 4: 价格类正则检查
    Step 5: LLM兜底分类（ClassifyAgent）
    Return: 意图标签 {'price'|'tech'|'default'|'no_reply'}
```

**关键词库**：
```python
技术类：['参数', '规格', '型号', '连接', '对比']
价格类：['便宜', '价', '砍价', '少点']
```

#### 3.2 Agent基类（BaseAgent）
```python
class BaseAgent:
    def generate(user_msg, item_desc, context, bargain_count):
        1. 构建消息链（System + User）
        2. 调用LLM生成回复
        3. 执行安全过滤
        4. 返回最终响应
```

#### 3.3 议价专家（PriceAgent）
```python
# 动态温度控制：鼓励后期降价幅度增加
temperature = 0.3 + min(bargain_count * 0.05, 0.25)  # 上限0.55

# 议价次数信息注入
system_prompt += f"\n▲当前议价轮次：{bargain_count}"

# 最后一次议价特殊处理
if bargain_count >= 3:
    temp = 0.55  # 温度最高，鼓励让步
```

#### 3.4 安全过滤器（Safety Filter）
```python
def _safe_filter(text):
    blocked_phrases = ["微信", "QQ", "支付宝", "银行卡", "线下"]
    if any phrase in text:
        return "[安全提醒]请通过平台沟通"
    return text
```

### 4. 上下文管理器 (`context_manager.py` - ChatContextManager)

**职责**：数据库驱动的对话历史和上下文管理

**核心功能**：
```python
# 对话历史操作
get_chat_context(chat_id, max_messages=100)
save_message(user_id, item_id, role, content, chat_id)
format_history(context) -> "user: msg1\nassistant: msg2\n..."

# 议价追踪
add_bargain_record(chat_id)
get_bargain_count(chat_id) -> int

# 商品缓存
save_item_info(item_id, item_data)
get_item_info(item_id) -> dict
```

**数据库表**：
- `messages`：完整对话记录（1:N关系）
- `chat_bargain_counts`：议价轮次统计
- `items`：商品信息缓存

### 5. Prompt模板系统

**路径**：`prompts/` 目录

**4个专家Prompt**：
```
prompts/
├── classify_prompt.txt          # 意图分类
├── classify_prompt_example.txt  # (备用)
├── price_prompt.txt             # 议价策略
├── price_prompt_example.txt     # (备用)
├── tech_prompt.txt              # 技术支持
├── tech_prompt_example.txt      # (备用)
├── default_prompt.txt           # 通用客服
└── default_prompt_example.txt   # (备用)
```

**加载优先级**：`*.txt` > `*_example.txt`

**Prompt内容示例**：
```yaml
# classify_prompt.txt
你是一个意图识别专家，根据用户消息、商品信息和对话历史，
判断用户意图属于以下哪一类：
- price: 议价/砍价请求
- tech: 技术/参数咨询
- default: 其他问题
- no_reply: 无需回复

# price_prompt.txt
你是闲鱼卖家的议价专家，根据议价轮次调整降价幅度：
- 第1轮（bargain_count=0）：保守，最多降5%
- 第2-3轮：逐步增加，最多降15%
- 第4轮+：灵活让步，根据市场价调整
```

---

## ⚡ 快速开始

### 前置条件
- Python 3.8+
- 闲鱼网页版登录账户（用于获取Cookie）
- 大语言模型API密钥（OpenAI兼容接口）

### 安装步骤

#### 1️⃣ 克隆仓库
```bash
git clone https://github.com/shaxiu/XianyuAutoAgent.git
cd XianyuAutoAgent
```

#### 2️⃣ 创建虚拟环境（推荐）
```bash
# Windows
python -m venv venv
venv\Scripts\activate

# Linux/Mac
python3 -m venv venv
source venv/bin/activate
```

#### 3️⃣ 安装依赖
```bash
pip install -r requirements.txt
```

#### 4️⃣ 配置环境变量

创建 `.env` 文件：
```env
# 必需配置
API_KEY=your_api_key_here                    # LLM API密钥
MODEL_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1  # 模型服务地址
MODEL_NAME=qwen-max                          # 模型名称
COOKIES_STR=your_cookies_here                # 闲鱼网页版Cookie

# 可选配置
HEARTBEAT_INTERVAL=15                        # 心跳间隔（秒）
TOKEN_REFRESH_INTERVAL=3600                  # Token刷新间隔（秒）
MANUAL_MODE_TIMEOUT=3600                     # 人工模式超时（秒）
TOGGLE_KEYWORDS=。                           # 模式切换关键词
SIMULATE_HUMAN_TYPING=False                  # 是否模拟人工输入延迟
```

**如何获取COOKIES_STR**：
1. 浏览器访问 https://www.goofish.com/
2. 按 `F12` 打开开发者工具 → `Network` 标签
3. 点击一个请求 → `Cookies` 部分
4. 复制所有Cookie内容，粘贴到 `COOKIES_STR=`

#### 5️⃣ 创建本地Prompt文件（可选）
```bash
# 不创建则自动使用_example模板
cp prompts/classify_prompt_example.txt prompts/classify_prompt.txt
cp prompts/price_prompt_example.txt prompts/price_prompt.txt
cp prompts/tech_prompt_example.txt prompts/tech_prompt.txt
cp prompts/default_prompt_example.txt prompts/default_prompt.txt
```

编辑prompt文件自定义AI行为：
```text
# Customize your prompts here
你是一个专业的闲鱼卖家客服...
```

#### 6️⃣ 运行应用
```bash
python main.py
```

**日志输出**：
```
[2026-03-13 10:00:00] ✅ 连接注册完成
[2026-03-13 10:01:23] 📨 新消息: 这个能便宜点吗?
[2026-03-13 10:01:25] 🎯 意图识别: price (议价)
[2026-03-13 10:01:26] 💬 Agent回复: 亲，这款已经很优惠了...
[2026-03-13 10:01:27] ✉️ 消息已发送
```

---

## 🐳 Docker容器化部署

### 方式一：Docker Compose（推荐）

#### 1️⃣ 配置docker-compose.yml
```yaml
version: '3.8'

services:
  xianyu-agent:
    build: .
    container_name: xianyu-agent
    environment:
      - API_KEY=${API_KEY}
      - MODEL_BASE_URL=${MODEL_BASE_URL}
      - MODEL_NAME=${MODEL_NAME}
      - COOKIES_STR=${COOKIES_STR}
      - HEARTBEAT_INTERVAL=15
      - TOKEN_REFRESH_INTERVAL=3600
    volumes:
      - ./data:/app/data                    # 数据库文件持久化
      - ./prompts:/app/prompts              # Prompt配置挂载
      - ./logs:/app/logs                    # 日志持久化
    restart: unless-stopped
    networks:
      - xianyu-network

networks:
  xianyu-network:
    driver: bridge
```

#### 2️⃣ 启动服务
```bash
# 复制.env文件
cp .env.example .env
# 编辑.env，填入真实API密钥和Cookie

# 启动容器
docker-compose up -d

# 查看日志
docker-compose logs -f xianyu-agent

# 停止服务
docker-compose down
```

### 方式二：直接Docker

#### 构建镜像
```bash
docker build -t xianyu-agent:latest .
```

#### 运行容器
```bash
docker run -d \
  --name xianyu-agent \
  -e API_KEY=your_key \
  -e COOKIES_STR=your_cookies \
  -e MODEL_NAME=qwen-max \
  -v $(pwd)/data:/app/data \
  -v $(pwd)/prompts:/app/prompts \
  xianyu-agent:latest
```

### 数据库持久化
- SQLite数据库文件存储在 `./data/chat_history.db`
- Docker容器需挂载 `-v ./data:/app/data` 保证数据持久化
- 定期备份数据库文件

---

## 📊 数据库架构详解

### SQLite数据库设计
```
chat_history.db
├── Table: messages
│   ├── PK: id (自增)
│   ├── FK: user_id (用户ID)
│   ├── FK: item_id (商品ID)
│   ├── FK: chat_id (会话ID)
│   ├── role: 'user'|'assistant'|'system'
│   ├── content: 消息文本
│   └── timestamp: 创建时间
│   
│   索引：
│   ├── idx_user_item(user_id, item_id)
│   ├── idx_chat_id(chat_id)
│   └── idx_timestamp(timestamp)
│
├── Table: chat_bargain_counts
│   ├── PK: chat_id (会话ID)
│   ├── count: 议价轮数
│   └── last_updated: 更新时间
│
└── Table: items
    ├── PK: item_id (商品ID)
    ├── data: JSON商品完整信息
    ├── price: 价格数值
    ├── description: 商品描述
    └── last_updated: 更新时间
```

### 查询性能优化
```sql
-- 快速查询最近100条对话
SELECT * FROM messages 
WHERE chat_id = ? 
ORDER BY timestamp DESC 
LIMIT 100;

-- 快速查询议价轮数
SELECT count FROM chat_bargain_counts 
WHERE chat_id = ?;

-- 快速查询商品信息（缓存）
SELECT data FROM items 
WHERE item_id = ? 
AND  datetime(last_updated) > datetime('now', '-1 day');
```

---

## 🔄 消息处理流程

```
WebSocket接收
    ↓
├─ 解析MessagePack格式
│
├─ 判断消息类型
│  ├─ 同步包消息 → 解密
│  ├─ 心跳响应 → 更新时间
│  └─ 系统消息 → 忽略
│
├─ 提取用户消息
│  ├─ user_id (买家ID)
│  ├─ toid (卖家ID)
│  ├─ item_id (商品ID)
│  ├─ message (用户输入)
│  └─ timestamp
│
├─ 加载上下文
│  └─ 从SQLite查询chat_id的完整对话历史 (最多100条)
│
├─ 检查模式
│  ├─ 若为切换关键词 → toggle_manual_mode()
│  ├─ 若为人工模式 → 跳过AI回复，等待人工输入
│  └─ 若为AI模式 → 继续
│
├─ 获取商品信息
│  ├─ 先查SQLite缓存
│  ├─ 若无缓存或过期 → 调用getItemInfo API
│  └─ 构建商品描述JSON
│
├─ AI智能回复
│  ├─ Step1: 意图路由 (IntentRouter.detect)
│  ├─ Step2: 选择专家Agent
│  ├─ Step3: 生成初步回复
│  ├─ Step4: 安全过滤
│  └─ Return: 最终回复文案
│
├─ 数据库保存（异步）
│  ├─ INSERT 用户消息 → messages表
│  ├─ INSERT AI回复 → messages表
│  ├─ UPDATE 议价计数 → chat_bargain_counts表
│  └─ Commit事务
│
└─ 发送消息
   └─ WebSocket.send(消息格式)到闲鱼IM
```

---

## 🧪 配置自定义示例

### 示例1：调整议价策略

编辑 `prompts/price_prompt.txt`：
```yaml
你是闲鱼的议价谈判专家。
根据以下策略处理砍价请求：

定价策略：
- 底线：原价的80%（坚决不让）
- 首次砍价：最多降8%，话术要温和
- 第2-3次：降幅递进，最多15%
- 第4次+：视为高意愿买家，可考虑20%左右

话术要点：
- 突出商品优势和低价优势
- 使用"亲"、"快下单呀"等亲近话语
- 多用数据对比，如"同款店铺卖XXX元"
- 最后一次让步时，强调"赠送好评10元红包"
```

### 示例2：自定义技术专家

编辑 `prompts/tech_prompt.txt`：
```yaml
你是产品技术咨询专家。

你正在帮助卖家回答关于[商品]的技术问题。

知识库：
- iPhone 15: A17 Pro芯片，6GB内存，48MP主摄
- iPad Air: M1芯片，8GB+256GB起，支持Apple Pencil 2代
- ...

回答规则：
1. 优先用数据和参数回答
2. 提供产品对比表格（若涉及对比）
3. 承认不确定的信息，建议询问官方客服
4. 最后加上"有其他问题可继续咨询"
```

### 示例3：启用人工输入延迟

编辑 `.env`：
```env
# 模拟2-5秒的自然输入延迟，提升用户体验
SIMULATE_HUMAN_TYPING=True
```

---

## 📝 日志监控

### 日志级别
```
DEBUG    - 详细调试信息（Prompt加载、Hook触发等）
INFO     - 关键事件（连接成功、意图识别、消息发送）
WARNING  - 警告信息（Cookie即将失效、网络延迟）
ERROR    - 错误信息（API调用失败、解析异常）
SUCCESS  - 成功标志（登录成功、Token刷新成功）
```

### 关键日志示例
```log
[INFO] 成功加载所有提示词
[INFO] 获取初始token...
[INFO] Token刷新成功
[INFO] 连接注册完成
[DEBUG] 已加载 price_prompt 提示词，路径: prompts/price_prompt.txt, 长度: 1234 字符
[INFO] 意图识别完成: price
[INFO] 议价次数: 2
[INFO] 消息已保存至数据库
[WARNING] Token即将过期，准备刷新...
```

---

## 🔐 安全性说明

### Cookie管理
- Cookie存储于 `.env` 文件（**本地密钥，不上传Git**）
- 支持自动检测Cookie过期，提示用户更新
- 定期清理重复Cookie，保持会话清洁

### API密钥安全
- 所有API密钥存储于 `.env` 文件，不硬编码
- Docker运行时不将密钥暴露在日志中
- 建议使用环境变量或密钥管理服务（Vault等）

### 内容过滤
- 自动检测违规词汇（微信、QQ、支付宝等）
- 违规消息自动替换为安全提示
- 支持自定义违规词库

### 隐私保护
- 对话数据存储于本地SQLite（不上传第三方）
- 支持删除特定对话记录或整个数据库
- 遵守GDPR等隐私法规

---

## 🚨 故障排除

### 常见问题

#### Q1: "Cookie已失效，请重新登陆"
**原因**：
- Cookie在浏览器中已过期
- Cookies_STR被修改或截断

**解决**：
1. 重新打开闲鱼网页版：https://www.goofish.com/
2. 按 `F12` → `Network` → 点击任何请求
3. 复制完整Cookie字符串
4. 更新 `.env` 中的 `COOKIES_STR`
5. 重启应用

#### Q2: "Token获取失败"
**原因**：
- 触发了闲鱼风控（RGV587）
- 设备指纹被检测为异常

**解决**：
1. 查看日志是否出现 `RGV587_ERROR`
2. 关闭应用，等待30分钟
3. 重新在浏览器登陆闲鱼，完成滑块验证
4. 复制新Cookie并更新 `.env`
5. 重启应用

#### Q3: "消息无法发送"
**原因**：
- WebSocket连接已断开
- 消息格式有误
- 商品ID无效

**解决**：
1. 检查 `docker-compose logs` 或 `main.py` 日志
2. 确认闲鱼网页版是否正常
3. 尝试强制重新连接： `docker-compose restart`

#### Q4: "意图识别不准"
**原因**：
- Prompt质量不高
- 关键词库不完整
- LLM模型选择不合适

**解决**：
1. 编辑 `prompts/classify_prompt.txt`，优化提示词
2. 添加更多关键词到 `IntentRouter.rules`
3. 尝试更强大的模型，如 `qwen-plus` → `qwen-max`
4. 增加few-shot示例在Prompt中

#### Q5: "数据库锁定或损坏"
**原因**：
- 多进程并发写入
- 容器异常关闭

**解决**：
1. 首先备份 `./data/chat_history.db`
2. 删除 `./data/chat_history.db`，程序会自动重建
3. 如需恢复数据，使用备份文件：
   ```bash
   cp ./data/chat_history.db.bak ./data/chat_history.db
   ```

---

## 📈 性能优化

### 数据库查询优化
```python
# ✅ Good: 使用索引查询
SELECT * FROM messages WHERE chat_id = ? ORDER BY timestamp DESC LIMIT 100;

# ❌ Bad: 全表扫描
SELECT * FROM messages WHERE content LIKE '%关键词%';

# ✅ Good: 定期清理过期数据
DELETE FROM messages WHERE datetime(timestamp) < datetime('now', '-30 days');
```

### LLM调用优化
```python
# ✅ 缓存商品信息（避免重复API调用）
if item_id in cache and cache_not_expired():
    item_info = cache[item_id]
else:
    item_info = get_item_info(item_id)
    cache_with_ttl(item_id, item_info, ttl=3600)

# ✅ 使用较低温度快速响应
temperature = 0.3  # 一致性高，速度快
# vs
temperature = 0.9  # 创意强，但速度慢
```

### 消息批处理
```python
# ✅ 批量INSERT而非逐条插入
messages_batch = [msg1, msg2, msg3, ...]
for msg in messages_batch:
    cursor.execute("INSERT INTO messages ...", msg)
conn.commit()  # 单次提交
```

---

## 🤝 配置最佳实践

### 生产环境检查清单
- ✅ 使用强密码保护 `.env` 文件
- ✅ 定期备份 `./data/chat_history.db`
- ✅ 启用Docker的 `restart: unless-stopped` 策略
- ✅ 配置日志轮转（避免日志文件过大）
- ✅ 监控Token刷新是否成功（检查日志）
- ✅ 定期更新依赖包（pip install -U）
- ✅ 在生产环境使用反向代理（如Nginx）
- ✅ 启用HTTPS/WSS加密传输

### 推荐配置参数
```env
# 对于小卖家（日均<50条消息）
HEARTBEAT_INTERVAL=20
TOKEN_REFRESH_INTERVAL=7200

# 对于中等卖家（日均50-200条消息）
HEARTBEAT_INTERVAL=15
TOKEN_REFRESH_INTERVAL=3600

# 对于大卖家或多商品（日均>200条消息）
HEARTBEAT_INTERVAL=10
TOKEN_REFRESH_INTERVAL=1800
```

---

## 📖 API文档

### 消息结构

#### 用户消息格式（输入）
```json
{
  "user_id": "2088xxxxxxx",
  "item_id": "6567890xxxxxxx",
  "content": "这个能便宜点吗?",
  "chat_id": "2088xxxxxxx_6567890xxxxxxx"
}
```

#### AI回复格式（输出）
```json
{
  "text": "亲，这款已经优化到最低价啦，再便宜的话就没利润了...",
  "intent": "price",
  "bargain_count": 2
}
```

### Python SDK使用示例

```python
from XianyuAgent import XianyuReplyBot
from context_manager import ChatContextManager

# 初始化
bot = XianyuReplyBot()
ctx_mgr = ChatContextManager()

# 获取对话历史
history = ctx_mgr.get_chat_context(chat_id="xxx")

# 生成回复
reply = bot.generate_reply(
    user_msg="这个能最低多少钱?",
    item_desc=json.dumps({"title": "iPhone 15", "price": "5999"}),
    context=history,
)

# 保存消息
ctx_mgr.save_message(
    user_id="2088xxx",
    item_id="6567890xxx",
    role="assistant",
    content=reply,
    chat_id="2088xxx_6567890xxx"
)
```

---

## 📚 扩展开发指南

### 添加新的Agent专家

1. **继承BaseAgent**
```python
from XianyuAgent import BaseAgent

class CustomAgent(BaseAgent):
    def generate(self, user_msg, item_desc, context, bargain_count=0):
        messages = self._build_messages(user_msg, item_desc, context)
        # 自定义逻辑
        return self.safety_filter(response)
```

2. **注册到XianyuReplyBot**
```python
def _init_agents(self):
    self.agents = {
        'custom': CustomAgent(self.client, self.custom_prompt, self._safe_filter),
        # ... 其他agents
    }
```

3. **添加意图路由规则**
```python
class IntentRouter:
    def detect(self, user_msg, item_desc, context):
        if 'custom_keyword' in user_msg:
            return 'custom'
        # ... 其他规则
```

### 自定义数据库表

在 `context_manager.py` 中扩展 `_init_db()` 方法：

```python
def _init_db(self):
    super()._init_db()  # 调用父类初始化
    
    cursor = sqlite3.connect(self.db_path).cursor()
    
    # 创建自定义表
    cursor.execute('''
    CREATE TABLE IF NOT EXISTS custom_data (
        id INTEGER PRIMARY KEY,
        chat_id TEXT,
        custom_field TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
    ''')
    
    cursor.execute('''
    CREATE INDEX IF NOT EXISTS idx_custom ON custom_data (chat_id)
    ''')
```

---

## 📄 许可证

本项目采用 **MIT License**，详见 [LICENSE](LICENSE) 文件。

**免责声明**：
- ⚠️ 本项目仅供学习交流使用
- ⚠️ 使用本项目等同于同意遵守闲鱼平台的服务条款
- ⚠️ 开发者对因本项目引起的任何后果不承担责任
- ⚠️ 禁止用于商业目的或违法用途

---


## ⭐ 致谢

感谢所有贡献者和使用者的支持！

如果这个项目对你有帮助，请点个 **Star** ⭐ 来支持我们继续维护～

```
          ⭐️
         ⭐⭐⭐
        ⭐⭐⭐⭐⭐
       ⭐⭐⭐⭐⭐⭐⭐
      感谢您的支持！
       
```

---

## 📝 更新日志

### 版本 2.0（当前）
- ✨ 多Agent智能路由系统
- ✨ SQLite数据库上下文管理
- ✨ 动态议价温度控制
- ✨ Docker多阶段构建优化
- 🐛 修复Cookie管理内存泄漏
- 📖 完整的架构文档

### 版本 1.0
- 基础WebSocket连接
- 单Agent通用回复
- 内存对话历史

---

**最后更新**：2026年3月13日

