📧 AutoTrack-AI: 智能求职邮件追踪助手📖 项目简介 (Introduction)AutoTrack-AI 是一个运行在 Mac 上的 Python 本地应用程序。它旨在通过 AI 自动化管理你的求职流程。它自动从 Gmail 获取邮件，利用本地部署的大语言模型 (Ollama) 分析邮件内容，判断是“申请确认”、“面试邀请”还是“拒信”，并提取关键信息（公司、职位、日期）。所有数据存储在本地 SQLite 数据库中，并提供可视化的统计分析面板。核心理念： 隐私优先（本地运行）、自动化、数据驱动求职。🛠 技术栈 (Tech Stack)核心语言: Python 3.10+邮件获取: Gmail API (OAuth 2.0)AI 推理: Ollama (推荐模型: Llama 3, Qwen 2.5, 或 Mistral)Embeddings (关联分析): bge-m3 或 nomic-embed-text (通过 Ollama 调用)数据库: SQLite (支持简单的向量存储)数据处理: Pandas, NumPy (用于计算余弦相似度)UI/统计面板: Streamlit (推荐) 或 Plotly工具库: google-api-python-client, requests, sqlalchemy⚡️ 工作流与架构 (Workflow & Architecture)系统分为四个主要模块：Fetcher (邮件抓取器):通过 Gmail API 增量获取未读邮件或特定标签的邮件。解析邮件的 Subject, Body, Sender, Date, Thread ID。Analyzer (AI 分析器):将邮件内容发送给本地 Ollama 接口。任务 1 - 精细分类: 使用 Few-Shot Prompt 将邮件分为 9 类 (技术面、HR面、拒信等)。任务 2 - 提取: 提取实体 (Company Name, Job Title, Platform，仅针对 application_received 类型）。Linker (智能关联层 - 核心逻辑):目标: 将新收到的后续邮件（如拒信、面试）精准关联到最初的“申请记录”上。策略 A (Thread ID): 优先匹配 Gmail 的会话 ID，这是最准确的。策略 B (Vector Semantic Search - 向量语义搜索): * 如果 Thread ID 匹配失败，计算新邮件的 Embedding 向量。与数据库中 status='application_received' 的记录进行 余弦相似度 (Cosine Similarity) 计算。优势: 能处理多语言环境（如用法语回复的拒信匹配英语的申请）及模糊语义。Dashboard (统计展示):展示漏斗图 (Funnel): 投递 -> 电话面 -> 技术面 -> Offer。计算 Ghost Rate (无回复率) 和各阶段转化率。💾 数据库设计 (Database Schema)Table: applicationsColumnTypeDescriptionidINTEGER PK自增 IDcompany_nameTEXT公司名称job_titleTEXT职位名称statusTEXT当前状态 (Label from AI)embeddingBLOB/ARRAY(新) 公司+职位的向量数据，用于后续匹配applied_dateDATETIME首次投递时间last_updateDATETIME最后状态更新时间Table: email_logsColumnTypeDescriptionidINTEGER PK自增 IDapplication_idFK关联到的 applications 表 IDcategory_labelTEXTAI 判断的类别 (e.g., interview_invitation_technical)raw_contentTEXT邮件部分内容created_atDATETIME收信时间🤖 AI Prompt (Classification)使用 Few-Shot Learning 提高准确率。You are a job-related email classification model.  
Classify each email into exactly one category.  
Follow the rules strictly.

### CATEGORIES
- application_received
- interview_invitation_phone
- interview_invitation_technical
- interview_invitation_hr
- interview_followup
- offer
- rejection
- advertisement_or_newsletter
- other

### RULES
- Output only the label.
- No explanation.
- No extra words.
- Choose the closest category.

### EXAMPLES

#### Example 1
EMAIL:
"Bonjour,
Notre équipe a bien reçu votre candidature pour être Data Engineer ...
Bonne journée
L'équipe de recrutement TOHTEM"
OUTPUT:
application_received

#### Example 2
EMAIL:
"Bonjour Daoyuan,
... Après étude attentive de votre candidature...
nous avons privilégié d’autres profils...
L'équipe DEV RH – Recrutement"
OUTPUT:
rejection

#### Example 3
EMAIL:
"Vous souhaitez intégrer une startup innovante...
Découvrez des startups à la recherche...
Networking, pitchs, inscriptions..."
OUTPUT:
advertisement_or_newsletter

#### Example 4
EMAIL:
"Hello Daoyuan,
Merci pour ta candidature...
Je te propose un premier échange téléphonique...
Benjamin de padoa"
OUTPUT:
interview_invitation_phone

### NOW CLASSIFY THE FOLLOWING EMAIL
EMAIL:
{{email_text}}

### OUTPUT FORMAT
label: <one category>
🚀 快速开始 (Getting Started)1. 环境准备安装 Ollama 并拉取 生成模型 和 Embedding 模型：# 用于分类和提取
ollama pull llama3 
# 或者使用 qwen (如果是中文环境较多)
ollama pull qwen2.5

# 用于向量关联 (推荐专门的 embedding 模型，比 LLM 更轻更快)
ollama pull bge-m3
2. Python 依赖pip install google-auth google-auth-oauthlib google-api-python-client pandas streamlit sqlalchemy numpy ollama
3. 核心逻辑伪代码 (Embeddings)import ollama
import numpy as np

def get_embedding(text):
    # 使用专门的 embedding 模型
    response = ollama.embeddings(model='bge-m3', prompt=text)
    return response['embedding']

def find_related_application(new_email_text, db_session):
    # 1. 计算新邮件的向量
    new_vec = get_embedding(new_email_text)
    
    # 2. 获取所有活跃申请的向量 (在实际应用中可以使用向量数据库优化)
    active_apps = db_session.query(Application).filter(Application.status == 'application_received').all()
    
    best_match = None
    highest_similarity = 0.0
    
    for app in active_apps:
        # 计算余弦相似度
        app_vec = np.array(app.embedding) # 假设存的是list
        similarity = np.dot(new_vec, app_vec) / (np.linalg.norm(new_vec) * np.linalg.norm(app_vec))
        
        if similarity > 0.85: # 设定一个阈值
            if similarity > highest_similarity:
                highest_similarity = similarity
                best_match = app
                
    return best_match
📝 LicenseMIT License
