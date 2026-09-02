
复刻指南：企业智能知识助手（制造行业 RAG 问答平台）



本文是一份完整的工程复刻文档。目标：任何 AI 或开发者读完本文，能从零复刻出一个可运行、可上线试点的企业知识库智能问答系统。

技术栈：Python 3.13 + FastAPI + SQLAlchemy(SQLite/可切PostgreSQL) + jieba BM25 + Qwen3-Embedding(本地) + OpenAI兼容LLM + 单文件原生JS前端。
总代码量约 2500 行，不含依赖安装约 2~4 小时可复刻完成。



0. 系统全景

                         ┌──────────────────────────────┐
                         │  浏览器（单文件 index.html）    │
                         │  聊天 / 知识库管理 / 管理后台   │
                         └──────────────┬───────────────┘
                                        │ HTTP + SSE
                         ┌──────────────▼───────────────┐
                         │  FastAPI  (backend/app/main) │
                         │  JWT 鉴权 → API 路由           │
                         └──┬──────────┬──────────┬─────┘
                            │          │          │
                 ┌──────────▼───┐ ┌────▼─────┐ ┌──▼──────────┐
                 │ 文档处理管线   │ │ RAG检索   │ │ 对话编排     │
                 │ 解析→切片→向量 │ │ BM25+向量 │ │ 流式LLM+落库 │
                 └──────┬───────┘ └────┬─────┘ └──┬──────────┘
                        │              │          │
        ┌───────────────▼──────────────▼──────────▼───────────┐
        │        SQLite (WAL)  —— 一库存全部业务数据            │
        │  users / knowledge_bases / documents / chunks        │
        │  conversations / messages / evaluation_dataset       │
        └──────────────────────────────────────────────────────┘
                     │                        │
          ┌──────────▼─────────┐   ┌──────────▼──────────────┐
          │ 本地 Qwen3-Embedding│   │ OpenAI兼容 LLM 网关      │
          │ (sentence-transformers)│ │ (zp/glm-5.3, 流式)      │
          └────────────────────┘   └─────────────────────────┘

核心设计原则（复刻时刻牢记）：





一个中间件都不多：不用 Redis/Celery/ES/Milvus/LangGraph。SQLite 起步，切 PostgreSQL 只改一个环境变量。



先鉴权后检索：在权限过滤后的候选集上建临时 BM25 索引，绝不允许"先搜后筛"。



LLM 可插拔：业务代码只依赖 llm_service，provider=mock 离线跑通全链路，provider=openai 接任意兼容接口，零代码切换。



AI 只用有效版本：文档带 version/status，同名文档上传新版自动把旧版置 expired，检索层强制 status=active。



不编造：检索为空时明确回答"知识库中没有找到相关依据"；回答带文档/版本/章节/页码引用。



模型故障可降级：Embedding 服务挂掉自动回退纯 BM25，问答不中断。





1. 目录结构（复刻时按此创建）

enterprise-ai/
├── backend/
│   ├── app/
│   │   ├── main.py                 # FastAPI 入口：建表/bootstrap admin/挂路由/托管前端
│   │   ├── models.py               # 全部 ORM 模型（单文件）
│   │   ├── core/
│   │   │   ├── config.py           # 环境变量配置 + .env 加载
│   │   │   ├── database.py         # engine/session，SQLite WAL pragma
│   │   │   └── security.py         # PBKDF2 口令哈希 + JWT + 依赖注入
│   │   ├── api/
│   │   │   ├── auth.py             # 登录/me/用户管理
│   │   │   ├── knowledge.py        # 知识库/文档/授权/切片查看
│   │   │   ├── chat.py             # SSE 流式问答/会话历史/评价
│   │   │   └── admin.py            # 统计/评测集
│   │   ├── rag/
│   │   │   ├── loader.py           # PDF/DOCX/XLSX/TXT/MD → Block(page,section,text)
│   │   │   ├── splitter.py         # 章节切片 + 滑窗兜底
│   │   │   └── retriever.py        # jieba BM25 索引（rank_bm25 实现）
│   │   └── services/
│   │       ├── document_service.py # 上传落盘→后台线程解析→入库→刷新索引
│   │       ├── rag_service.py      # 权限过滤→混合检索→引用组装
│   │       ├── embedding_service.py# local/http 双模式 Embedding
│   │       ├── llm_service.py      # LLM 网关 + 系统提示词 + mock 实现
│   │       └── chat_service.py     # SSE 事件流编排 + 消息落库
│   ├── tests/test_core.py          # 10 个端到端测试
│   ├── scripts/backfill_embeddings.py  # 存量切片回填向量
│   └── requirements.txt
├── frontend/index.html             # 单文件前端（约500行，原生JS）
├── deploy/                         # Dockerfile / docker-compose.yml / nginx.conf
├── sample-knowledge/               # 11 份示例规范文档（见 §9）
├── .env                            # 密钥与开关（勿提交）
└── .env.example





2. 数据模型（models.py，一次建对）

七张表，全部建在单文件里。复刻时注意每个字段的用途注释：

# users: id, username(unique), password_hash, real_name, department, role(admin|user), status(active), created_at
# knowledge_bases: id, name(unique), description, is_public(bool), created_at
# kb_permissions: id, kb_id(FK), user_id(FK), UNIQUE(kb_id,user_id)   # 私有库授权
# documents: id, kb_id(FK), name(展示名=文件名去扩展名), file_name, file_type,
#            file_path, version("V1.0"), document_type, status(active|expired),
#            parse_status(pending|processing|ready|failed), parse_error,
#            created_by(FK users), created_at, updated_at
# document_chunks: id, document_id(FK,index), content(Text), section,
#            page_number, chunk_index, embedding(Text,JSON浮点数组), created_at
# conversations: id, user_id(FK), title(取首问前50字), created_at, updated_at
# messages: id, conversation_id(FK,index), role(user|assistant), content,
#           citations(JSON引用列表), sources(JSON检索到的chunk+分数，评测用),
#           latency_ms, rating(1|-1|0), rating_comment, created_at
# evaluation_dataset: id, question, expected_answer, expected_source, created_at

关键设计决策：





embedding 存 JSON 文本而不是 pgvector——SQLite 兼容、百万级以内可用；切 PostgreSQL 后可平滑迁 pgvector。



citations 和 sources 分开存：citations 给前端展示，sources（含 chunk id 与检索分数）留给评测脚本算召回率。



messages.rating 存在消息行上，统计时 GROUP BY rating 即可算满意度。

数据库初始化（database.py）要点：

engine = create_engine(url, connect_args={"check_same_thread": False} if sqlite else {})
# SQLite 必须开 WAL + 外键：
@event.listens_for(engine, "connect")
def _pragma(dbapi_conn, _):
    cur = dbapi_conn.cursor()
    cur.execute("PRAGMA journal_mode=WAL"); cur.execute("PRAGMA foreign_keys=ON"); cur.close()





3. 配置与安全

config.py 三件事：





启动时 load_dotenv(项目根/.env) —— 注意路径用 Path(__file__).resolve().parents[3]（config 在 backend/app/core/ 下三层才是项目根。这里我们踩过坑：少算一层导致 .env 读不到、数据库建到错误目录）。



所有配置项 os.getenv 带默认值，关键项：SECRET_KEY / DATABASE_URL / LLM_PROVIDER(mock|openai) / LLM_BASE_URL / LLM_API_KEY / LLM_MODEL / EMBEDDING_ENABLED / EMBEDDING_PROVIDER(local|http) / EMBEDDING_LOCAL_MODEL / CHUNK_SIZE=500 / CHUNK_OVERLAP=80 / TOP_K=8 / RETRIEVE_CANDIDATES=30。



BASE_DIR 指向 backend/，UPLOAD_DIR = BASE_DIR/data/uploads，按 kb_id/uuid后缀.扩展名 存文件。

security.py：PBKDF2-HMAC-SHA256 12 万次迭代 + 随机盐（格式 pbkdf2$iters$salt$hex）；JWT payload {sub, role, exp} HS256；FastAPI 依赖 get_current_user（解 Bearer token→查用户→校验 status）和 require_admin。

main.py 启动逻辑：

@app.on_event("startup")
def startup():
    Base.metadata.create_all(engine)
    # 引导管理员：users 表空则创建 admin（密码来自 ADMIN_BOOTSTRAP_PASSWORD）
    # 调 reindex(db)：全量 chunk 重建 BM25 内存索引
# 路由前缀：/api/auth + /api + /api + /api/admin（注意 auth/admin 各自带子前缀，
# 否则登录路径会变成 /api/login —— 我们踩过这个坑）
# GET /api/health 返回 {"status","llm_provider","embedding"} 供探活
# 最后：如果 ../frontend/index.html 存在，@app.get("/") 用 FileResponse 返回它





4. RAG 核心三件套



4.1 loader.py —— 解析

统一输出 Block(page, section, text)：





PDF（PyMuPDF）：逐页取 page.get_text("dict")，用字号判定标题（≥14pt 或 ≥12.5pt 且行内一致、长度<50）更新当前 section，记录页码。



DOCX（python-docx）：按文档流顺序迭代段落与表格；style.name.startswith("Heading") 取标题层级维护标题栈，section 输出为 标题1 > 标题2 路径；表格转 单元格 | 单元格 行。



XLSX（openpyxl read_only + data_only）：首行作表头，每行输出 列名:值; 列名:值 保住列语义，section 为 Sheet[x]数据，相邻行合并到 ~3000 字符。



TXT/MD：# 行作 section；注意 UTF-8 失败时回退 GB18030（制造业老文档常见）。

blocks_to_markdown(blocks) 把 Block 序列拼回文本，section 变成 ## 标题 行——这是与 splitter 的衔接协议。

4.2 splitter.py —— 切片





split_markdown()：按 ##  行切开，得到 [{section, text}]。



chunk_sections()：章节体 ≤CHUNK_SIZE(500) 直接成片；超长则滑窗——先按句号/问号/分号/换行切句，句子累积超限就落一片，下一片带上前一句尾部 overlap(80) 保证上下文连续；每个 chunk 的最终文本 = f"{section}\n{正文}"（section 前缀提升检索命中）。



4.3 retriever.py —— BM25

class BM25Index:
    def build(items: list[tuple[int, str]]):  # jieba.lcut 分词 + 去停用词 + rank_bm25.BM25Okapi
    def search(query, top_k) -> list[tuple[int, float]]  # 过滤 0 分

jieba 分词时 lower() 并过滤虚词停用词表（"的了和是就对不与以及或对于通过根据按照…"）。





5. Embedding 服务（双模式 + 降级）

# EMBEDDING_PROVIDER=local: sentence_transformers.SentenceTransformer(本地模型目录) 进程内加载
#                           懒加载 + threading.Lock 单例；encode(normalize_embeddings=True)
# EMBEDDING_PROVIDER=http:  POST {EMBEDDING_BASE_URL}/embeddings，OpenAI 格式，按 index 排序取 embedding
# Qwen3 系列支持查询侧 instruct 前缀：EMBEDDING_QUERY_INSTRUCT 配置后 embed_query 时加前缀，文档侧不加
# EMBEDDING_ENABLED=false:  embed_texts 返回 []，整条链路自动变纯 BM25

实测参数：Qwen3-Embedding-0.6B 在 CPU 上加载约 8 秒、单条约 1~2 秒、输出 1024 维。语义自检脚本（复刻后先跑这个验证模型方向正确）：

"故障怎么处理" 对 "E103表示液压压力异常…" 相似度应 > 对 "液压油保养…" > 对 "食堂菜单…"
实测：0.7585 > 0.3956 > 0.2267





6. 检索与问答编排



6.1 rag_service.search() —— 每一步都有讲究

def search(db, user, question, kb_ids=None, top_k=8):
    # 1) 权限：allowed = 公开库 ∪ 用户被授权的私有库；传入 kb_ids 与 allowed 求交集
    # 2) 候选：join documents 过滤 kb_id IN (allowed) AND status='active' AND parse_status='ready'
    #    ← "AI 只用当前有效版本"在这一条 WHERE 里实现
    # 3) 在候选集上建临时 BM25Index 并 search ← "先鉴权后检索"的实现方式
    # 4) EMBEDDING_ENABLED 时 _fuse_vectors()：
    #    fused = (bm25/max_bm25)*(1-w) + cosine(qvec, chunk_vec)*w，w=0.6
    #    向量相似度 <0.2 的丢弃；embed_query 抛任何异常 → return 纯 BM25 结果（降级）
    # 5) 组装结果 [{id,text,doc_id,doc_name,version,section,page,score}]



6.2 llm_service.py —— 网关与提示词

系统提示词（这是"不胡说"的关键，直接抄）：

你是企业内部智能助手。回答问题时必须遵守：
1. 优先使用提供的企业知识库资料。
2. 不得编造企业内部制度、参数、流程。
3. 如果知识库中没有相关资料，明确告诉用户"知识库中没有找到相关依据"。
4. 对设备参数、工艺参数、质量标准不得自行推测。
5. 对存在多个版本的文档，优先使用当前有效版本。
6. 回答涉及企业制度时必须给出资料来源。
7. 回答涉及设备维修时，需要提示安全注意事项。
8. 回答末尾给出参考文档。

上下文块格式（片段编号+来源行，mock 模式也按此解析）：

[片段1] 来源：《文档名 V2.3》 章节名 第47页
正文……

流式解析（OpenAI SSE）：data: 行 → JSON → choices[0].delta.content。思考型模型（如 glm-5.3）会先发 reasoning_content 再发 content，只取 content 即可自动跳过思考。非流式同理取 message.content。LLM_TIMEOUT 设 180 秒（思考模型首 token 慢，实测整答约 10 秒）。

mock 实现：从上下文里正则抽出片段列表拼回复——离线开发/CI 时没有模型也能全链路联调，这个投资回报极高。

6.3 chat_service.ask_stream() —— SSE 事件协议

event: meta   data: {"conversation_id", "citations":[{doc_id,doc_name,version,section,page}], "retrieved":n}
event: delta  data: {"text": "增量"}        ← n 次
event: done   data: {"conversation_id","message_id","latency_ms","citations"}
event: error  data: {"message"}

顺序：建/取会话 → 检索 → 先落库用户消息（含 sources JSON） → 发 meta → 逐 delta → 落库助手消息（citations+latency） → done。历史携带最近 3 轮（6 条消息）拼进 messages。

FastAPI 侧：StreamingResponse(gen, media_type="text/event-stream", headers={"Cache-Control":"no-cache","X-Accel-Buffering":"no"})。

前端 fetch reader 手动按 \n\n 分帧解析（EventSource 不支持 POST）。





7. 文档处理管线与 API



7.1 上传即解析（异步）

POST /api/documents/upload（multipart：kb_id/file/version/document_type）：





校验扩展名白名单 .{pdf,docx,xlsx,txt,md}；



落盘 UPLOAD_DIR/kb_id/uuid12.后缀，登记 documents 行（parse_status=pending）；



同名文档全部置 expired（不含本次新 doc）——版本自动下线；



threading.Thread(target=process_document, daemon=True) 后台执行；



立即返回 202 + 文档 JSON（前端轮询 parse_status）。

process_document（用独立 SessionLocal）：置 processing → 解析→切片→（可选）向量化→旧 chunk 全删→新 chunk 入库→ready/failed(+parse_error)。完成后 refresh_index() 全量重建全局 BM25 索引（MVP 规模毫秒级，够用）。

7.2 API 一览

POST /api/auth/login                GET /api/auth/me
GET  /api/auth/users                POST /api/auth/users          (admin)
GET  /api/knowledge-bases           POST /api/knowledge-bases     (admin)
DELETE /api/knowledge-bases/{id}    POST/DELETE /api/knowledge-bases/{id}/grants/{user_id}
GET  /api/knowledge-bases/{id}/documents
POST /api/documents/upload (202)    PATCH /api/documents/{id}/status?new_status=active|expired
DELETE /api/documents/{id}          GET  /api/documents/{id}/chunks   (切片审核)
POST /api/chat/stream (SSE)         GET  /api/conversations
GET  /api/conversations/{id}/messages  DELETE /api/conversations/{id}
POST /api/messages/{id}/rating      {rating:1|-1|0, comment}
GET  /api/admin/statistics          GET/POST /api/admin/eval/items
GET  /api/health

踩坑记录（复刻时避免）：





admin.py 统计接口里 db.query(Message.content).all() 返回的是 Row 元组列表，直接塞 JSON 会序列化爆炸——必须 [row[0] for row in ...]。



Pydantic 的 created_at 字段类型要写 datetime | None 而不是 str | None，否则 ORM 对象校验失败。



BM25Index.__init__ 不要带位置参数，构造处两处调用要一致。





8. 前端（单文件 index.html）

零构建、零依赖、约 500 行。布局：左侧栏（品牌/新建对话/会话列表/三个导航）+ 主区三视图。

必做的交互细节：





登录：token 存 localStorage，启动时 GET /api/auth/me 静默续期，401 清 token 回登录页。



SSE 解析：fetch + resp.body.getReader() + TextDecoder，按 \n\n 切帧，event: 行取类型、data: 行 JSON.parse；meta 事件记 citations 和 conversation_id，delta 追加到气泡（尾部加 ▌光标），done 后渲染引用与👍👎按钮。



知识库管理：管理员可见建库/上传/下线/删除按钮；上传后 loadDocs(); setTimeout(loadDocs, 2500); setTimeout(loadDocs, 6000) 应对异步解析。



检索范围：顶栏下拉选知识库 → kb_ids 传给 chat/stream。



管理后台：统计卡片（用户/提问数/有效文档/切片数/满意度）、用户表+创建用户、评测集录入表格。



所有用户内容渲染前 escapeHtml。



9. 示例知识库内容（可选但强烈推荐）

sample-knowledge/ 下 11 份 Markdown 规范（每份 30~60 条可执行条款，标题用 ## 第X章）：设备维修保养管理规范（三级保养500h/2500h/8000h、润滑五定、点检闭环、MTBF/MTTR）、车间安全生产管理规范（动火/受限空间/高处作业 GB 30871-2022、LOTO、三级教育24/72学时）、6S现场管理规范（颜色划线标准）、质量检验管理规范（GB/T 2828.1-2012 AQL 抽样与转移规则、首件三检）、注塑机安全操作规程（GB/T 22530-2022、油温15~55℃、恒温15~30min防冷启动断螺杆），以及 6 份人事制度（考勤加班/请假休假/入职试用/绩效奖惩/离职交接/宿舍食堂，依据劳动法与劳动合同法，公司自定数值用行业常见口径）。

上传方式：建"规章制度库"后循环走 upload API（见 §7.2），等 parse_status=ready。





10. 测试（10 个端到端用例，复刻后必须全绿）

tests/test_core.py，TestClient 跑真实 app。在 import app 之前用环境变量指到临时 SQLite 和临时上传目录，保证测试隔离。





错密码登录 401；2. 无 token 访问 me 401；3. 建用户+重名 409；



切片器：章节切分正确 + 超长文本触发滑窗产生多片；



md 解析含关键词；



全链路：上传 md 文档→轮询 ready→chunks≥2→提问命中 E103（SSE 含 meta/done）→无关问题触发"没有找到相关依据"；



版本下线：同名 SOP 传 V1.0 再传 V2.0→V1.0 变 expired、检索不到旧内容；



私有库权限：未授权用户列表不可见/documents 403/检索 retrieved=0→授权后可见；



非管理员上传 403；



评价落库→统计接口满意度/文档数/切片数正确→评测集增查。

已知耗时：全量约 60 秒（embedding 开启时每个解析文档都过一遍本地模型），CI 里可设 EMBEDDING_ENABLED=false 加速。

11. 部署

deploy/：Dockerfile（python:3.12-slim，VOLUME /data）+ docker-compose（api + nginx:alpine，前端静态挂载，/api/ 反代 proxy_buffering off; proxy_read_timeout 300s; client_max_body_size 50m——SSE 不关缓冲就不流式）+ nginx.conf。环境变量全透传，SECRET_KEY 上线必改。

12. 实际运行数据（复刻后可对照）





启动：uvicorn 8000 端口，健康检查 {"llm_provider":"openai","embedding":true}



混合检索实测排序（w=0.6）：语义问法"注塑机报液压压力不正常的故障如何排查"→ 维修手册 0.762/0.489 居前，无关文档被压到 0.26 以后；关键词问法"E103怎么处理"→ 0.831/0.735



glm-5.3 思考型模型整答延迟约 10 秒（156 条切片语料、TOP_K=8）



156 条存量切片回填耗时约 3 分钟（CPU）；新上传文档解析+向量化约 1~2 秒/条



6 个制度类问题验收（年休假/试用期离职/迟到处理/动火作业/注塑机开机检查/二级保养）全部命中正确文档与条款



13. 上线前检查清单





换强 SECRET_KEY 与 admin 密码；.env 加入 .gitignore



文档解析质量抽检（切片预览接口）；扫描件 PDF 当前不支持（会报"解析结果为空"），需补 OCR 管线



收集 20~100 道真实员工问题录入评测集（/api/admin/eval/items），作为后续调参回归基线



每个知识库指定业务侧知识管理员；版本更新流程 = 新版本上传自动下线旧版



思考型模型较慢，如需提速换非思考模型或网关侧关闭思考



日志按企业数据安全要求做脱敏与保留策略



未来演进方向：Reranker 精排（Qwen3-Reranker-0.6B）、Query Rewrite、页码级"查看原文"跳转、企业微信集成、Agent 工具调用（MES/ERP 只读查询）



14. 复刻顺序建议（可执行的施工序）





models.py + database.py + config.py（半天内可跑通建表）



security.py + auth.py → 用 curl 完成 login/me（第一个里程碑）



loader/splitter + document_service + knowledge.py → 上传一份 md 等 ready，查 chunks



retriever + rag_service.search → 直接 python 脚本调 search() 验证命中（第二个里程碑）



llm_service（先 mock）+ chat_service + chat.py → curl SSE 看到 mock 回答



前端 index.html → 浏览器全流程走通（第三个里程碑）



接真实 LLM（.env 改 provider=openai）



embedding_service + backfill 脚本 + 混合检索（第四个里程碑）



admin 统计/评测集 → 跑 10 个测试全绿 → deploy 部署文件

每一步都有可验证的产出，任何一步卡住都不要跳步——前一步是后一步的地基。
