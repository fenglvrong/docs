# 企业 AI 知识库基础版复刻实施手册

> 文档版本：V1.0  
> 整理日期：2026-09-02  
> 目标读者：负责复刻、维护或升级本项目的 AI 编程助手与开发人员

## 1. 文档目的

本文记录本次“传统制造企业内部 AI 知识库 + 智能问答机器人”从方案收敛到可运行基础版的完整实现方式。新的 AI 读取本文后，应能在 Windows 或 Linux 环境中复刻出同等能力，并清楚区分：

- 当前已经实现且验证过的能力；
- 为降低初期复杂度采用的基础版方案；
- 可以按阶段替换的模型和基础设施；
- 已经遇到并解决的关键问题；
- 尚未达到生产级的边界。

本文不得包含真实 API Key、正式管理员密码或企业敏感资料。复制项目时必须重新生成密钥并按环境注入。

## 2. 产品定位与实施原则

项目名称：企业知识助手基础版。

第一阶段只完成以下闭环：

```text
登录 → 上传文档 → 解析 → 切片 → 向量化 → 混合召回
     → Reranker 重排 → 大模型回答 → 引用原文 → 保存历史与反馈
```

基础版不直接接入 MES、ERP、WMS、OA，不执行设备控制或业务写操作。这样可以先验证员工是否愿意使用、知识是否能被检索、答案是否可追溯，再决定是否扩展 Agent。

实现原则：

1. 知识库没有依据时明确拒答，不让模型自行编造企业制度、工艺参数或维修方法。
2. 回答必须返回来源文档、页码或章节、切片摘要。
3. 模型接入均通过可替换的服务层，业务接口不绑定单一厂商。
4. 模型、数据库、存储、权限都允许从基础方案平滑升级。
5. 法规、标准和内部制度必须标记版本、适用范围、来源和审核状态。

## 3. 当前技术栈

| 层 | 当前实现 | 说明 |
| --- | --- | --- |
| 前端 | Vue 3 + TypeScript + Vite | 单页聊天、文档管理、历史会话和统计 |
| 后端 | Python 3.12 + FastAPI | REST API 与 SSE 流式回答 |
| 数据库 | SQLite | 保存文档、切片、向量、会话和反馈 |
| 文件存储 | 本地目录 | `data/uploads` |
| 文档解析 | pypdf + python-docx | 支持 PDF、DOCX、TXT、Markdown |
| LLM | OpenAI 兼容 Chat Completions | 当前接入 GLM 兼容模型 |
| Embedding | Sentence Transformers | 可选本地 Qwen3 Embedding |
| Reranker | Transformers CausalLM | 可选本地 Qwen3 Reranker |
| 部署 | 本地进程或 Docker Compose | 本地开发端口为 5173 和 8001 |

### 3.1 模型作为可选升级项

最小版本在不配置任何模型时也能运行：使用稳定哈希向量完成检索，并把相关切片直接作为答案。模型可以按下面顺序升级，而不必一次全部安装。

| 能力 | 基础方案 | 推荐升级项 |
| --- | --- | --- |
| LLM | 本地摘录式回答 | 任意 OpenAI 兼容大模型；当前验证过 `zp/glm-5.3` |
| Embedding | 384 维稳定哈希向量 | `Qwen/Qwen3-Embedding-0.6B`，1024 维 |
| Reranker | 不启用，按混合召回分数排序 | `Qwen/Qwen3-Reranker-0.6B` |

也可用用户要求的简表表达：

| 组件 | 升级模型 |
| --- | --- |
| Embedding | Qwen3-Embedding |
| Reranker | Qwen3-Reranker |

## 4. 项目目录

```text
enterprise-ai-basic/
├── backend/
│   ├── app/
│   │   ├── api/                    # 登录、文档、检索、聊天、会话、统计
│   │   ├── services/               # 解析、切片、Embedding、检索、重排、LLM
│   │   ├── config.py               # 环境变量配置
│   │   ├── database.py             # SQLite 表结构和连接
│   │   ├── main.py                 # FastAPI 入口和健康检查
│   │   ├── schemas.py              # API 数据结构
│   │   └── security.py             # 基础管理员认证和 HMAC Token
│   ├── scripts/
│   │   └── reindex_embeddings.py   # 更换 Embedding 后重建向量
│   ├── tests/                       # 当前两个轻量单元测试
│   ├── .env.example
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/
│   ├── src/
│   │   ├── App.vue                 # 聊天、文档、历史、统计主界面
│   │   ├── api.ts                  # API 与 SSE 客户端
│   │   ├── main.ts
│   │   └── style.css
│   ├── nginx/default.conf
│   ├── vite.config.ts
│   ├── package.json
│   └── Dockerfile
├── data/
│   ├── generated/                  # 自动生成并保留来源的内部制度草案
│   ├── models/                     # 本地模型，禁止提交版本库
│   ├── uploads/                    # 已上传原始文件
│   └── knowledge.db                # SQLite 数据库
├── docs/
├── docker-compose.yml
└── README.md
```

## 5. 从零复刻步骤

### 5.1 创建后端环境

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
```

后端依赖：

```text
fastapi==0.116.1
uvicorn[standard]==0.35.0
python-multipart==0.0.20
httpx==0.28.1
pypdf==6.0.0
python-docx==1.2.0
sentence-transformers==5.1.0
```

`sentence-transformers` 会带入 PyTorch、Transformers 和 Hugging Face 相关依赖。生产构建建议锁定完整依赖树并使用企业内部制品库。

### 5.2 创建前端环境

```powershell
cd frontend
npm install
npm run dev -- --host 127.0.0.1
```

主要版本：Vue 3.5、Vite 7、TypeScript 5.9。

### 5.3 配置环境变量

示例配置如下，真实密钥必须放在本地 `.env` 或密钥管理系统中：

```dotenv
APP_NAME=企业知识助手
APP_DATA_DIR=../data
MAX_UPLOAD_MB=20
CHUNK_SIZE=800
CHUNK_OVERLAP=120
TOP_K=5
MIN_RETRIEVAL_SCORE=0.30

ADMIN_USERNAME=admin
ADMIN_PASSWORD=replace-with-a-strong-password
TOKEN_SECRET=replace-with-a-long-random-secret

LLM_API_BASE=https://your-openai-compatible-gateway/v1
LLM_API_KEY=replace-with-secret
LLM_MODEL=your-chat-model

# 远程 Embedding，使用本地模型时可留空。
EMBEDDING_API_BASE=
EMBEDDING_API_KEY=
EMBEDDING_MODEL=

# 本地 Embedding 升级项。
LOCAL_EMBEDDING_MODEL=Qwen/Qwen3-Embedding-0.6B
LOCAL_EMBEDDING_MODEL_PATH=../data/models/Qwen3-Embedding-0.6B
LOCAL_EMBEDDING_DEVICE=cpu

# 本地 Reranker 升级项。
LOCAL_RERANKER_MODEL=Qwen/Qwen3-Reranker-0.6B
LOCAL_RERANKER_MODEL_PATH=../data/models/Qwen3-Reranker-0.6B
LOCAL_RERANKER_DEVICE=cpu
RERANK_CANDIDATE_COUNT=8
RERANK_MAX_LENGTH=1024
```

模型优先级：本地 Embedding 配置存在时优先使用本地模型；否则尝试远程 Embedding；两者都未配置时使用哈希向量。Reranker 未配置时直接返回混合召回排序结果。

### 5.4 下载本地模型

建议把模型放到 `data/models`，不要提交 Git：

```powershell
.\backend\.venv\Scripts\python.exe -c "from huggingface_hub import snapshot_download; snapshot_download('Qwen/Qwen3-Embedding-0.6B', local_dir='data/models/Qwen3-Embedding-0.6B')"
.\backend\.venv\Scripts\python.exe -c "from huggingface_hub import snapshot_download; snapshot_download('Qwen/Qwen3-Reranker-0.6B', local_dir='data/models/Qwen3-Reranker-0.6B')"
```

当前 Reranker 权重文件 `model.safetensors` 的实测信息：

```text
字节数：1191588280
SHA256：27cd75a405b9c1b46b59abfd88aaa209e6fed2a1972cde9b70e7659537c5e65b
```

模型仓库内容可能随上游更新变化，复刻时应固定 revision，并记录实际下载版本和哈希。

### 5.5 启动服务

本机 8000 端口已被其他服务使用，因此本项目开发环境使用 8001：

```powershell
cd backend
.\.venv\Scripts\python.exe -m uvicorn app.main:app --host 127.0.0.1 --port 8001 --env-file .env
```

另开终端：

```powershell
cd frontend
npm run dev -- --host 127.0.0.1
```

访问地址：

- 前端：`http://127.0.0.1:5173/`
- 后端：`http://127.0.0.1:8001/`
- Swagger：`http://127.0.0.1:8001/docs`
- 健康检查：`http://127.0.0.1:8001/api/health`

Vite 的 `/api` 代理必须指向 `http://127.0.0.1:8001`。本项目同时存在 `vite.config.ts` 和 TypeScript 编译生成的 `vite.config.js`，两者必须保持一致；否则 Vite 可能优先读取旧的 JavaScript 文件并把登录请求错误地代理到 8000。

## 6. 数据模型

基础版使用以下 SQLite 表：

### 6.1 documents

保存文档名称、原始文件名、类型、本地路径、SHA256、状态、切片数量和上传时间。`file_hash` 唯一，用于阻止相同内容重复上传。

### 6.2 document_chunks

保存文档切片正文、页码、章节、顺序和 JSON 格式向量。删除文档时级联删除切片。

### 6.3 conversations 与 chat_messages

保存当前用户的历史对话、用户消息、助手消息、引用来源和模型模式。

### 6.4 answer_feedback

保存“有帮助/没帮助”、原因、备注和更新时间。同一用户对同一助手消息只保留一条最新评价。

基础版只有单管理员账号，没有独立用户表、部门表、知识库表和权限表。

## 7. 文档入库链路

```text
POST /api/documents/upload
  → 校验扩展名与文件大小
  → 流式保存文件并计算 SHA256
  → 按 SHA256 检查重复内容
  → PDF / DOCX / TXT / Markdown 解析
  → 按段落切分，超长内容滑动窗口切分
  → Qwen3 Embedding 或备用向量生成
  → 在一个数据库事务中写入文档和切片
```

当前切片参数：

| 参数 | 值 |
| --- | ---: |
| `CHUNK_SIZE` | 800 字符 |
| `CHUNK_OVERLAP` | 120 字符 |
| 最大上传文件 | 20 MB |
| Embedding 维度 | Qwen3 为 1024；哈希备用为 384 |

切换 Embedding 模型后，旧向量不可继续混用。执行：

```powershell
cd backend
.\.venv\Scripts\python.exe scripts\reindex_embeddings.py
```

## 8. 检索与 Reranker 链路

```text
用户问题
  → 生成查询向量
  → 遍历当前有效切片并计算余弦相似度
  → 提取中英文检索词并计算关键词覆盖率
  → 混合分数 = 0.75 × 向量相似度 + 0.25 × 关键词覆盖率
  → MIN_RETRIEVAL_SCORE 阈值过滤
  → 截取前 RERANK_CANDIDATE_COUNT 个候选
  → Qwen3 Reranker 二次排序
  → 返回 Top K
```

基础版将向量保存在 SQLite JSON 字段中并在 Python 内遍历计算，只适合小规模试用。文档或并发增长后应替换为 pgvector 或专用向量库。

### 8.1 Qwen3 Reranker 的关键实现

不能把 `Qwen/Qwen3-Reranker-0.6B` 当作普通 `SentenceTransformers CrossEncoder` 直接加载。实测 `sentence-transformers 5.1.0` 会构造未训练的 `Qwen3ForSequenceClassification` 分类头，并出现无 Padding Token 等问题；即使勉强运行，分类分数也不可信。

正确实现采用模型官方推荐的生成式判别方式：

1. 使用 `AutoTokenizer` 和 `AutoModelForCausalLM`。
2. 使用官方 system/query/document 指令结构。
3. 在回答位置读取 `yes` 和 `no` 两个 Token 的 logits。
4. 对两个 logits 做 Softmax，得到 0～1 的相关概率。
5. 使用左侧 Padding；Tokenizer 没有 Padding Token 时回退到 EOS Token。
6. 模型使用线程锁延迟加载，异步接口通过 `asyncio.to_thread` 避免直接阻塞事件循环。

直接构造两个候选验证时，“E103 维修内容”得分 1.0，“人事休假内容”得分 0.0，排序正确。E103 并不在当前知识库文档中，因此通过完整检索接口提问 E103 时会在重排前被 0.30 阈值过滤，这是正确的拒答行为，不是 Reranker 故障。

### 8.2 CPU 性能边界

本机实测：

| 场景 | 耗时 |
| --- | ---: |
| 两个短候选，模型冷加载 | 约 8 秒 |
| 两个短候选，模型热运行 | 约 1.2 秒 |
| 8 个真实长切片完整检索 | 约 24～31 秒 |

因此 CPU 配置适合验证正确性，不适合高并发生产。可优先调整候选数、最大长度并建立评测集，再决定是否使用 GPU 或独立推理服务。

## 9. 问答链路

```text
POST /api/chat 或 POST /api/chat/stream
  → 创建或确认会话归属
  → 保存用户问题
  → 执行检索和重排
  → 无来源时直接回答“知识库中没有找到相关依据”
  → 有来源且未配置 LLM 时返回检索摘录
  → 有来源且已配置 LLM 时调用 OpenAI 兼容接口
  → 返回答案、来源、模型模式
  → 保存助手消息
```

系统提示词明确要求：只能根据提供的企业资料回答，资料不足时拒绝猜测，回答简洁可执行，并用 `[来源1]` 格式标注引用。

当前 LLM 配置已通过最小 Chat Completions 请求验证，配置名为 `zp/glm-5.3`，中转服务实际返回模型标识 `glm-5.3`。某些推理模型会先输出 `reasoning_content`，过小的 `max_tokens` 可能在正式答案出现前被截断；项目正式流式请求未主动设置该小限制。

## 10. API 清单

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| GET | `/api/health` | 服务状态及当前模型模式 |
| POST | `/api/auth/login` | 管理员登录 |
| GET | `/api/documents` | 文档列表 |
| POST | `/api/documents/upload` | 上传、解析、切片和建索引 |
| GET | `/api/documents/{id}/file` | 查看原始文档 |
| DELETE | `/api/documents/{id}` | 删除文档及其切片 |
| POST | `/api/knowledge/search` | 只执行知识检索 |
| POST | `/api/chat` | 非流式问答 |
| POST | `/api/chat/stream` | SSE 流式问答 |
| GET | `/api/conversations` | 历史会话列表 |
| GET | `/api/conversations/{id}/messages` | 会话消息 |
| DELETE | `/api/conversations/{id}` | 删除会话 |
| POST | `/api/feedback` | 保存答案评价 |
| GET | `/api/admin/statistics` | 文档、问答、未命中和满意度统计 |

除健康检查和登录外，其余接口使用 Bearer Token。当前 Token 为服务端 HMAC 签名，默认有效期 8 小时。

## 11. 前端实现要点

当前前端包含：

- 登录页；
- 新建、选择和删除历史对话；
- 文档上传、列表和删除；
- SSE 流式回答；
- 引用卡片及原文打开；
- 有帮助/没帮助评价；
- 文档数、提问数、未命中数和满意度统计。

布局使用固定视口高度。桌面端左侧栏固定，历史对话和文档列表在侧栏内部滚动，右侧消息独立滚动；长回答不会继续拉长左侧栏。关键 CSS 约束为：

```css
html, body, #app { height: 100%; }
body { overflow: hidden; }
.app-shell { height: 100dvh; min-height: 0; overflow: hidden; }
.sidebar { height: 100dvh; min-height: 0; overflow: hidden; }
.document-list { min-height: 0; flex: 1; overflow-y: auto; }
.chat-panel { height: 100dvh; min-height: 0; overflow: hidden; }
.messages { min-height: 0; overflow-y: auto; }
```

移动端媒体查询恢复整页滚动。浏览器实测视口高 720 像素时，页面总高度、侧栏和聊天区均保持 720 像素；消息区和文档列表分别在内部滚动。

## 12. 自动生成知识文档

基础版已经演示“检索公开权威来源 → 生成内部草案 → 上传并向量化”的流程。原则如下：

1. 优先使用政府网站、国家标准全文公开系统等权威来源。
2. 只引用标准题录和允许公开的信息，不批量复制受版权保护的标准全文。
3. 自动生成内容统一标注“内部试行草案”，不得冒充国家标准或法律原文。
4. 每份文档写明版本、生成日期、适用范围、审核要求、来源链接和核验日期。
5. 涉及劳动、职业健康、安全、环保等事项时，必须提示由 HR、法务、安全或合规负责人审核。
6. 地方差异明显的病假工资、婚产假、最低工资等内容不凭模型记忆硬填数值。

当前已经生成并入库 7 份草案：

1. 制造企业机械设备安全与点检维护规范；
2. 制造企业职业健康安全与环境管理规范；
3. 制造企业质量检验与不合格品控制规范；
4. 员工入职试用转岗与离职管理制度；
5. 员工考勤休假与加班管理制度；
6. 员工培训岗位资格与绩效沟通制度；
7. 生产交接班与现场人员行为规范。

当前数据库状态为 7 份有效文档、21 个切片。提问“工作满十年的员工每年有多少天年休假”时，首条正确命中《员工考勤休假与加班管理制度》，Reranker 得分约 0.9831。

## 13. 验证清单与已知结果

### 13.1 后端检查

```powershell
cd backend
.\.venv\Scripts\python.exe -m compileall -q app
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
```

当前两个定向单元测试均通过：

- 切片长度、页码和章节保留；
- 备用哈希向量的稳定性和归一化。

按照项目约束，不要每次简单改动都跑全量测试；根据改动范围执行定向测试。新建方法按项目代码风格添加中文注释。

### 13.2 运行检查

健康接口应包含：

```json
{
  "status": "ok",
  "llm_mode": "your-chat-model",
  "embedding_mode": "Qwen/Qwen3-Embedding-0.6B",
  "reranker_mode": "Qwen/Qwen3-Reranker-0.6B"
}
```

至少验证以下场景：

- 正确密码可通过前端 5173 的代理登录，不只测试直连后端；
- 上传支持的文件后能够生成切片；
- 相关问题返回正确文档；
- 无关问题在阈值过滤后返回零来源；
- 点击引用可以打开原始文档；
- 长回答只滚动右侧消息区；
- 历史对话、删除和反馈可正常使用。

## 14. 本次实施中解决的关键问题

### 14.1 端口冲突

本机 8000 已由其他服务占用，没有停止该服务。本项目本地后端改用 8001，Vite 代理同步指向 8001。Docker 容器内部仍可使用 8000，由 Compose 负责映射。

### 14.2 前端密码看似错误

后端直连 `8001/api/auth/login` 实际登录成功，但前端通过 5173 登录返回 401。原因不是密码，而是 `vite.config.ts` 已改为 8001、编译生成的 `vite.config.js` 仍指向 8000，运行中的 Vite 读取了旧配置。修复方法是同步两个配置并重启 Vite。复刻项目时最好只保留一个源码配置，避免漂移。

### 14.3 Embedding 切换后旧向量不可用

从哈希向量切换到 Qwen3 Embedding 后，向量维度和空间发生变化。必须重建全部现有切片向量，否则检索会出现维度不一致或结果失真。

### 14.4 Reranker 模型类型错误

Qwen3 Reranker 不是普通二分类 CrossEncoder。错误加载会出现随机分类头和 Padding Token 问题。必须使用 CausalLM 的 `yes/no` logits 方案。

### 14.5 阈值在 Reranker 之前生效

当前流程先按混合召回分数执行 `MIN_RETRIEVAL_SCORE`，再进入重排。若相关内容的初始召回分数低于阈值，Reranker 没有机会补救。调参时必须同时观察召回率、拒答率和重排效果，不能只看最终 Reranker 分数。

## 15. Docker 说明

```powershell
Copy-Item backend\.env.example backend\.env
docker compose up --build
```

容器访问地址为 `http://127.0.0.1:8080`。Nginx 将 `/api/` 转发到 Compose 内的 `backend:8000`。

当前 Dockerfile 没有把本地大模型复制进镜像，Compose 也只挂载了 `/data`。若启用本地 Qwen 模型，需要确认：

- `APP_DATA_DIR=/data`；
- `LOCAL_*_MODEL_PATH` 使用容器内路径，例如 `/data/models/...`；
- 主机 `./data` 已挂载到容器 `/data`；
- 镜像包含模型运行所需的 PyTorch CPU 或 GPU 运行环境；
- GPU 场景增加 NVIDIA Container Toolkit 和设备声明。

## 16. 生产升级路线

| 能力 | 当前基础版 | 推荐升级 |
| --- | --- | --- |
| 数据库 | SQLite | PostgreSQL |
| 向量检索 | Python 全量遍历 | pgvector；数据量更大时评估 Milvus |
| 全文检索 | 轻量关键词覆盖率 | Elasticsearch / OpenSearch BM25 |
| 文件存储 | 本地目录 | MinIO / 对象存储 |
| 异步任务 | 上传请求内同步处理 | Celery / RQ + Redis |
| 文档解析 | pypdf、python-docx | PyMuPDF、PaddleOCR、表格结构化解析 |
| 身份认证 | 单管理员 HMAC Token | LDAP / AD / OIDC / 企业微信 |
| 权限 | 登录后全库可见 | 用户、部门、角色、知识库和文档级 RBAC |
| 文档治理 | 文件级 active 状态 | 逻辑文档、版本、生效、失效、审批、密级 |
| Embedding | 哈希或 Qwen3-Embedding | GPU 推理服务、批处理、向量版本管理 |
| Reranker | 本地 Qwen3-Reranker | GPU 服务化、动态候选数、超时和降级 |
| LLM | 单 OpenAI 兼容模型 | 模型网关、主备路由、成本和配额控制 |
| 评测 | 人工问答验证 | 固定评测集、Recall@K、引用准确率、拒答率 |
| 监控 | 健康接口和本地日志 | Prometheus、Grafana、Loki/ELK、链路追踪 |
| Agent | 未实现 | 只读查询工具 → 审批式写操作；高风险操作必须人工确认 |

## 17. 生产化前必须补齐

1. 账号、部门、角色和文档级权限，并在检索前过滤权限。
2. 文档版本、审批、生效日期、失效日期和历史版本查询。
3. 上传病毒扫描、文件类型真实性校验、敏感信息识别和审计。
4. PostgreSQL + pgvector、对象存储、异步索引和失败重试。
5. OCR、复杂表格、图片说明、Excel 结构化处理和解析预览。
6. Prompt Injection 防护：上传内容只能作为资料，不能覆盖系统指令。
7. 模型调用超时、熔断、降级、限流、成本统计和数据出境评估。
8. 真实业务评测集，覆盖设备、安全、质量、生产和人事问题。
9. 日志脱敏、数据保留周期、备份恢复和灾难演练。
10. Agent 接入 MES/ERP/WMS/OA 时只通过受控 API；任何高风险写操作均需权限校验和人工确认。

## 18. 推荐给后续 AI 的执行顺序

后续 AI 接手时按以下顺序工作：

1. 阅读本文、`README.md`、`.env.example` 和现有代码，不读取或输出真实密钥。
2. 检查 5173、8001 端口和 `/api/health`，确认当前运行状态。
3. 明确本次请求是分析、修复还是升级，不擅自扩大范围。
4. 修改模型前先确认向量维度、已有索引和重建方案。
5. 修改检索前建立至少一组“应命中”和“应拒答”的问题。
6. 修改前端后在真实浏览器验证，不只依赖构建成功。
7. 只运行与改动相关的单元测试；简单业务改动可不补测试，不默认跑全量测试。
8. 新建 Python 或 TypeScript 方法时添加符合当前代码风格的中文注释。
9. 最终交付必须区分源码确认、测试确认、浏览器确认和生产部署确认。

## 19. 当前完成状态

截至 2026-09-02：

- 前端 5173、后端 8001 可访问；
- GLM OpenAI 兼容接口已实际连通；
- Qwen3 Embedding 已安装并用于文档向量化；
- Qwen3 Reranker 已安装、校验并进入检索链路；
- 文档上传、删除、检索、问答、引用、会话、反馈和统计可用；
- 7 份自动生成的制造业及人员制度草案已入库，共 21 个切片；
- 左侧栏和右侧回答区域已改为桌面端独立滚动；
- 当前仍是单机基础版，尚未完成企业级权限、文档治理、高可用和生产安全加固。

