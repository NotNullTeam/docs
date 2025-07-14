# 后端开发与AI模型团队任务指南

> **团队规模**: 5人 (原后端3人 + 原模型2人)
> **核心职责**: 搭建服务架构、实现混合检索算法、知识库管理、API开发、AI模型集成、RAG流程优化
> **技术栈**: Python + Flask、MySQL、SQLAlchemy、RQ、Docker、JWT认证、阿里云文档智能、langgraph、Weaviate、Qwen3系列模型、OLLAMA

## 📋 任务概览

### 主要任务清单

#### 🏗️ 基础架构任务 (后端主导)
- [x] **任务1**: Flask项目搭建与认证系统 (预计2-3天)
- [ ] **任务2**: 数据库设计与模型定义 (预计2-3天)
- [ ] **任务3**: 核心业务API实现 (预计5-6天)
- [ ] **任务4**: 知识文档管理API (预计3-4天)
- [ ] **任务5**: 异步任务处理系统 (预计3-4天)
- [ ] **任务6**: 数据看板API实现 (预计2-3天)

#### 🤖 AI能力任务 (模型主导)
- [ ] **任务7**: 数据收集与IDP集成 (预计3-4天)
- [ ] **任务8**: 向量数据库构建与配置 (预计2-3天)
- [ ] **任务9**: 混合检索算法实现 (预计4-5天)
- [ ] **任务10**: 提示词工程与优化 (预计3-4天)
- [ ] **任务11**: langgraph Agent流程构建 (预计5-6天)
- [ ] **任务12**: 模型接口集成与测试 (预计2-3天)

#### 🔧 集成测试任务 (全团队)
- [ ] **任务13**: 系统集成与端到端测试 (预计3-4天)
- [ ] **任务14**: (进阶)小模型微调 (预计5-7天，可选)

---

## 🎯 任务1: Flask项目搭建与认证系统

### 1.1 项目初始化 (第1天)

**目标**: 创建Flask项目基础结构，配置开发环境。

**具体步骤**:

1. **创建项目目录结构**
   ```bash
   mkdir ip-expert-backend
   cd ip-expert-backend
   
   # 创建目录结构
   mkdir -p {app/{api,models,services,utils},config,migrations,tests}
   touch app/__init__.py app/api/__init__.py app/models/__init__.py
   ```

2. **创建虚拟环境**
   ```bash
   python -m venv venv
   source venv/bin/activate  # Linux/Mac
   # 或 venv\Scripts\activate  # Windows
   ```

3. **安装依赖包** (`requirements.txt`)
   ```txt
   Flask==2.3.3
   Flask-SQLAlchemy==3.0.5
   Flask-Migrate==4.0.5
   Flask-JWT-Extended==4.5.2
   Flask-CORS==4.0.0
   Flask-WTF==1.1.1
   marshmallow==3.20.1
   redis==4.6.0
   rq==1.15.1
   PyMySQL==1.1.0
   python-dotenv==1.0.0
   requests==2.31.0

   # AI服务相关依赖 - 统一使用Langchain集成
   langchain>=0.2.0
   langchain-community>=0.2.0
   langchain_openai>=0.1.0
   dashscope>=1.14.1

   # 阿里云文档智能服务
   alibabacloud_tea_openapi
   alibabacloud_docmind_api20220711==1.4.7
   alibabacloud_credentials

   # 向量数据库
   weaviate-client==3.25.3
   ```

4. **创建应用工厂** (`app/__init__.py`)
   ```python
   from flask import Flask
   from flask_sqlalchemy import SQLAlchemy
   from flask_migrate import Migrate
   from flask_jwt_extended import JWTManager
   from flask_cors import CORS
   from config import Config
   
   db = SQLAlchemy()
   migrate = Migrate()
   jwt = JWTManager()
   
   def create_app(config_class=Config):
       app = Flask(__name__)
       app.config.from_object(config_class)
       
       # 初始化扩展
       db.init_app(app)
       migrate.init_app(app, db)
       jwt.init_app(app)
       CORS(app)
       
       # 注册蓝图
       from app.api import bp as api_bp
       app.register_blueprint(api_bp, url_prefix='/api/v1')
       
       return app
   ```

**验收标准**:
- Flask应用能正常启动
- 项目结构清晰合理
- 依赖包安装完成

### 1.2 配置管理 (第1天)

**目标**: 设置配置文件和环境变量管理。

**具体步骤**:

1. **创建配置文件** (`config.py`)
   ```python
   import os
   from dotenv import load_dotenv
   
   basedir = os.path.abspath(os.path.dirname(__file__))
   load_dotenv(os.path.join(basedir, '.env'))
   
   class Config:
       SECRET_KEY = os.environ.get('SECRET_KEY') or 'dev-secret-key'
       
       # 数据库配置
       SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
           'mysql+pymysql://root:password@localhost/ip_expert'
       SQLALCHEMY_TRACK_MODIFICATIONS = False
       
       # JWT配置
       JWT_SECRET_KEY = os.environ.get('JWT_SECRET_KEY') or 'jwt-secret'
       JWT_ACCESS_TOKEN_EXPIRES = 3600  # 1小时
       
       # Redis配置
       REDIS_URL = os.environ.get('REDIS_URL') or 'redis://localhost:6379'
       
       # 文件上传配置
       UPLOAD_FOLDER = os.environ.get('UPLOAD_FOLDER') or 'uploads'
       MAX_CONTENT_LENGTH = 16 * 1024 * 1024  # 16MB
   ```

2. **创建环境变量文件** (`.env`)
   ```bash
   SECRET_KEY=your-secret-key-here
   JWT_SECRET_KEY=your-jwt-secret-here
   DATABASE_URL=mysql+pymysql://root:password@localhost/ip_expert
   REDIS_URL=redis://localhost:6379

   # AI服务相关 - Langchain统一集成配置
   DASHSCOPE_API_KEY=your-dashscope-api-key
   OPENAI_API_KEY=your-dashscope-api-key  # 用于langchain_openai兼容接口
   OPENAI_API_BASE=https://dashscope.aliyuncs.com/compatible-mode/v1

   # 阿里云文档智能服务
   ALIBABA_ACCESS_KEY_ID=your-access-key-id
   ALIBABA_ACCESS_KEY_SECRET=your-access-key-secret

   # OLLAMA本地重排序模型服务
   OLLAMA_BASE_URL=http://localhost:11434
   ```

**验收标准**:
- 配置文件结构合理
- 环境变量正确加载
- 支持不同环境配置

### 1.3 JWT认证系统 (第2天)

**目标**: 实现基于JWT的用户认证系统。

**具体步骤**:

1. **创建用户模型** (`app/models/user.py`)
   ```python
   from app import db
   from werkzeug.security import generate_password_hash, check_password_hash
   from datetime import datetime
   
   class User(db.Model):
       id = db.Column(db.Integer, primary_key=True)
       username = db.Column(db.String(80), unique=True, nullable=False)
       email = db.Column(db.String(120), unique=True, nullable=False)
       password_hash = db.Column(db.String(255), nullable=False)
       roles = db.Column(db.String(100), default='user')
       created_at = db.Column(db.DateTime, default=datetime.utcnow)
       is_active = db.Column(db.Boolean, default=True)
       
       def set_password(self, password):
           self.password_hash = generate_password_hash(password)
           
       def check_password(self, password):
           return check_password_hash(self.password_hash, password)
           
       def to_dict(self):
           return {
               'id': self.id,
               'username': self.username,
               'email': self.email,
               'roles': self.roles.split(',') if self.roles else []
           }
   ```

2. **实现认证API** (`app/api/auth.py`)
   ```python
   from flask import Blueprint, request, jsonify
   from flask_jwt_extended import create_access_token, jwt_required, get_jwt_identity
   from app.models.user import User
   from app import db
   
   bp = Blueprint('auth', __name__)
   
   @bp.route('/login', methods=['POST'])
   def login():
       data = request.get_json()
       username = data.get('username')
       password = data.get('password')
       
       user = User.query.filter_by(username=username).first()
       
       if user and user.check_password(password) and user.is_active:
           access_token = create_access_token(identity=user.id)
           return jsonify({
               'access_token': access_token,
               'token_type': 'Bearer',
               'user_info': user.to_dict()
           })
       
       return jsonify({'error': '用户名或密码错误'}), 401
   
   @bp.route('/me', methods=['GET'])
   @jwt_required()
   def get_current_user():
       user_id = get_jwt_identity()
       user = User.query.get(user_id)
       return jsonify(user.to_dict())
   ```

**验收标准**:
- 用户登录功能正常
- JWT token生成和验证正确
- 受保护的API需要认证

---

## 🎯 任务2: 数据库设计与模型定义

### 2.1 数据库设计 (第1天)

**目标**: 设计完整的数据库表结构。

**具体步骤**:

1. **核心表设计**
   - users: 用户表
   - cases: 诊断案例表
   - nodes: 节点表
   - edges: 边表
   - knowledge_documents: 知识文档表
   - parsing_jobs: 解析任务表
   - feedback: 反馈表

2. **创建数据库** (MySQL)
   ```sql
   CREATE DATABASE ip_expert CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   ```

3. **启动MySQL服务** (Docker)
   ```yaml
   # docker-compose.local.yml 中添加
   mysql:
     image: mysql:8.0
     ports:
       - "3306:3306"
     environment:
       MYSQL_ROOT_PASSWORD: password
       MYSQL_DATABASE: ip_expert
     volumes:
       - mysql_data:/var/lib/mysql
   ```

**验收标准**:
- 数据库设计文档完成
- MySQL服务正常运行
- 数据库连接测试通过

### 2.2 SQLAlchemy模型定义 (第2天)

**目标**: 定义所有数据库模型。

**具体步骤**:

1. **案例相关模型** (`app/models/case.py`)
   ```python
   from app import db
   from datetime import datetime
   import uuid
   
   class Case(db.Model):
       __tablename__ = 'cases'
       
       id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
       title = db.Column(db.String(200), nullable=False)
       status = db.Column(db.Enum('open', 'solved', 'closed'), default='open')
       user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
       created_at = db.Column(db.DateTime, default=datetime.utcnow)
       updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
       
       # 关系
       nodes = db.relationship('Node', backref='case', lazy='dynamic', cascade='all, delete-orphan')
       edges = db.relationship('Edge', backref='case', lazy='dynamic', cascade='all, delete-orphan')
   
   class Node(db.Model):
       __tablename__ = 'nodes'
       
       id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
       case_id = db.Column(db.String(36), db.ForeignKey('cases.id'), nullable=False)
       type = db.Column(db.Enum('USER_QUERY', 'AI_ANALYSIS', 'AI_CLARIFICATION', 'USER_RESPONSE', 'SOLUTION'))
       title = db.Column(db.String(200))
       status = db.Column(db.Enum('COMPLETED', 'AWAITING_USER_INPUT', 'PROCESSING'), default='PROCESSING')
       content = db.Column(db.JSON)
       metadata = db.Column(db.JSON)
       created_at = db.Column(db.DateTime, default=datetime.utcnow)
   
   class Edge(db.Model):
       __tablename__ = 'edges'
       
       id = db.Column(db.Integer, primary_key=True)
       case_id = db.Column(db.String(36), db.ForeignKey('cases.id'), nullable=False)
       source = db.Column(db.String(36), nullable=False)
       target = db.Column(db.String(36), nullable=False)
   ```

2. **知识文档模型** (`app/models/knowledge.py`)
   ```python
   class KnowledgeDocument(db.Model):
       __tablename__ = 'knowledge_documents'
       
       id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
       filename = db.Column(db.String(255), nullable=False)
       original_filename = db.Column(db.String(255), nullable=False)
       file_path = db.Column(db.String(500), nullable=False)
       file_size = db.Column(db.Integer)
       mime_type = db.Column(db.String(100))
       
       # 元数据
       vendor = db.Column(db.String(50))
       tags = db.Column(db.JSON)
       
       # 处理状态
       status = db.Column(db.Enum('QUEUED', 'PARSING', 'INDEXED', 'FAILED'), default='QUEUED')
       progress = db.Column(db.Integer, default=0)
       error_message = db.Column(db.Text)
       
       # 时间戳
       uploaded_at = db.Column(db.DateTime, default=datetime.utcnow)
       processed_at = db.Column(db.DateTime)
       
       user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
   
   class ParsingJob(db.Model):
       __tablename__ = 'parsing_jobs'
       
       id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
       document_id = db.Column(db.String(36), db.ForeignKey('knowledge_documents.id'), nullable=False)
       status = db.Column(db.Enum('PENDING', 'PROCESSING', 'COMPLETED', 'FAILED'), default='PENDING')
       started_at = db.Column(db.DateTime)
       completed_at = db.Column(db.DateTime)
       error_message = db.Column(db.Text)
       result_data = db.Column(db.JSON)
       
       created_at = db.Column(db.DateTime, default=datetime.utcnow)
   ```

**验收标准**:
- 所有模型定义完成
- 关系映射正确
- 数据库迁移脚本生成成功

### 2.3 数据库迁移 (第3天)

**目标**: 创建和执行数据库迁移。

**具体步骤**:

1. **初始化迁移**
   ```bash
   flask db init
   flask db migrate -m "Initial migration"
   flask db upgrade
   ```

2. **创建初始数据** (`scripts/init_data.py`)
   ```python
   from app import create_app, db
   from app.models.user import User
   
   def create_admin_user():
       app = create_app()
       with app.app_context():
           admin = User(
               username='admin',
               email='admin@example.com',
               roles='admin,user'
           )
           admin.set_password('admin123')
           db.session.add(admin)
           db.session.commit()
           print("Admin user created successfully")
   
   if __name__ == '__main__':
       create_admin_user()
   ```

**验收标准**:
- 数据库表创建成功
- 迁移脚本正常工作
- 初始管理员用户创建成功

---

## 🎯 任务3: 核心业务API实现

### 3.1 案例管理API (第1-2天)

**目标**: 实现诊断案例的CRUD操作。

**具体步骤**:

1. **案例列表API** (`app/api/cases.py`)
   ```python
   from flask import Blueprint, request, jsonify
   from flask_jwt_extended import jwt_required, get_jwt_identity
   from app.models.case import Case, Node, Edge
   from app import db

   bp = Blueprint('cases', __name__)

   @bp.route('/cases', methods=['GET'])
   @jwt_required()
   def get_cases():
       user_id = get_jwt_identity()

       # 获取查询参数
       status = request.args.get('status')
       vendor = request.args.get('vendor')
       category = request.args.get('category')

       query = Case.query.filter_by(user_id=user_id)

       if status:
           query = query.filter_by(status=status)

       cases = query.order_by(Case.updated_at.desc()).all()

       return jsonify([{
           'caseId': case.id,
           'title': case.title,
           'status': case.status,
           'createdAt': case.created_at.isoformat() + 'Z',
           'updatedAt': case.updated_at.isoformat() + 'Z'
       } for case in cases])
   ```

2. **创建案例API**
   ```python
   @bp.route('/cases', methods=['POST'])
   @jwt_required()
   def create_case():
       user_id = get_jwt_identity()
       data = request.get_json()

       query = data.get('query')
       attachments = data.get('attachments', [])

       if not query:
           return jsonify({'error': '问题描述不能为空'}), 400

       # 创建案例
       case = Case(
           title=query[:100] + '...' if len(query) > 100 else query,
           user_id=user_id
       )
       db.session.add(case)
       db.session.flush()  # 获取case.id

       # 创建用户问题节点
       user_node = Node(
           case_id=case.id,
           type='USER_QUERY',
           title='用户问题',
           status='COMPLETED',
           content={
               'text': query,
               'attachments': attachments
           }
       )
       db.session.add(user_node)
       db.session.flush()

       # 创建AI分析节点
       ai_node = Node(
           case_id=case.id,
           type='AI_ANALYSIS',
           title='AI分析中...',
           status='PROCESSING'
       )
       db.session.add(ai_node)
       db.session.flush()

       # 创建边
       edge = Edge(
           case_id=case.id,
           source=user_node.id,
           target=ai_node.id
       )
       db.session.add(edge)

       db.session.commit()

       # 触发异步AI分析任务
       from app.services.agent_service import analyze_user_query
       analyze_user_query.delay(case.id, ai_node.id, query)

       return jsonify({
           'caseId': case.id,
           'title': case.title,
           'nodes': [
               {
                   'id': user_node.id,
                   'type': user_node.type,
                   'title': user_node.title,
                   'status': user_node.status,
                   'content': user_node.content
               },
               {
                   'id': ai_node.id,
                   'type': ai_node.type,
                   'title': ai_node.title,
                   'status': ai_node.status
               }
           ],
           'edges': [
               {
                   'source': edge.source,
                   'target': edge.target
               }
           ]
       })
   ```

**验收标准**:
- 案例CRUD操作正常
- 支持查询参数过滤
- 返回数据格式符合API文档

### 3.2 多轮交互API (第2-3天)

**目标**: 实现Agent多轮对话交互。

**具体步骤**:

1. **交互处理API**
   ```python
   @bp.route('/cases/<case_id>/interactions', methods=['POST'])
   @jwt_required()
   def handle_interaction(case_id):
       user_id = get_jwt_identity()
       data = request.get_json()

       case = Case.query.filter_by(id=case_id, user_id=user_id).first()
       if not case:
           return jsonify({'error': '案例不存在'}), 404

       parent_node_id = data.get('parentNodeId')
       response_data = data.get('response')
       retrieval_weight = data.get('retrievalWeight', 0.7)
       filter_tags = data.get('filterTags', [])

       # 创建用户响应节点
       user_response_node = Node(
           case_id=case_id,
           type='USER_RESPONSE',
           title='用户补充信息',
           status='COMPLETED',
           content=response_data
       )
       db.session.add(user_response_node)
       db.session.flush()

       # 创建AI处理节点
       ai_processing_node = Node(
           case_id=case_id,
           type='AI_ANALYSIS',
           title='AI分析中...',
           status='PROCESSING'
       )
       db.session.add(ai_processing_node)
       db.session.flush()

       # 创建边
       edge1 = Edge(case_id=case_id, source=parent_node_id, target=user_response_node.id)
       edge2 = Edge(case_id=case_id, source=user_response_node.id, target=ai_processing_node.id)
       db.session.add_all([edge1, edge2])

       db.session.commit()

       # 触发异步处理
       from app.services.agent_service import process_user_response
       process_user_response.delay(
           case_id,
           ai_processing_node.id,
           response_data,
           retrieval_weight,
           filter_tags
       )

       return jsonify({
           'newNodes': [
               {
                   'id': user_response_node.id,
                   'type': user_response_node.type,
                   'title': user_response_node.title,
                   'status': user_response_node.status,
                   'content': user_response_node.content
               },
               {
                   'id': ai_processing_node.id,
                   'type': ai_processing_node.type,
                   'title': ai_processing_node.title,
                   'status': ai_processing_node.status
               }
           ],
           'newEdges': [
               {'source': edge1.source, 'target': edge1.target},
               {'source': edge2.source, 'target': edge2.target}
           ],
           'processingNodeId': ai_processing_node.id
       })
   ```

2. **节点详情API**
   ```python
   @bp.route('/cases/<case_id>/nodes/<node_id>', methods=['GET'])
   @jwt_required()
   def get_node_detail(case_id, node_id):
       user_id = get_jwt_identity()

       case = Case.query.filter_by(id=case_id, user_id=user_id).first()
       if not case:
           return jsonify({'error': '案例不存在'}), 404

       node = Node.query.filter_by(id=node_id, case_id=case_id).first()
       if not node:
           return jsonify({'error': '节点不存在'}), 404

       return jsonify({
           'id': node.id,
           'type': node.type,
           'title': node.title,
           'status': node.status,
           'content': node.content,
           'metadata': node.metadata
       })
   ```

**验收标准**:
- 多轮交互流程正常
- 节点状态更新及时
- 异步任务正确触发

### 3.3 反馈处理API (第3天)

**目标**: 实现用户反馈收集和处理。

**具体步骤**:

1. **反馈模型** (`app/models/feedback.py`)
   ```python
   class Feedback(db.Model):
       __tablename__ = 'feedback'

       id = db.Column(db.String(36), primary_key=True, default=lambda: str(uuid.uuid4()))
       case_id = db.Column(db.String(36), db.ForeignKey('cases.id'), nullable=False)
       user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)

       outcome = db.Column(db.Enum('solved', 'unsolved'), nullable=False)
       comment = db.Column(db.Text)
       corrected_solution = db.Column(db.JSON)

       # 审核状态
       review_status = db.Column(db.Enum('pending', 'approved', 'rejected'), default='pending')
       reviewed_by = db.Column(db.Integer, db.ForeignKey('user.id'))
       reviewed_at = db.Column(db.DateTime)

       created_at = db.Column(db.DateTime, default=datetime.utcnow)
   ```

2. **反馈API**
   ```python
   @bp.route('/cases/<case_id>/feedback', methods=['POST'])
   @jwt_required()
   def submit_feedback(case_id):
       user_id = get_jwt_identity()
       data = request.get_json()

       case = Case.query.filter_by(id=case_id, user_id=user_id).first()
       if not case:
           return jsonify({'error': '案例不存在'}), 404

       feedback = Feedback(
           case_id=case_id,
           user_id=user_id,
           outcome=data.get('outcome'),
           comment=data.get('comment'),
           corrected_solution=data.get('corrected_solution')
       )

       db.session.add(feedback)

       # 更新案例状态
       if data.get('outcome') == 'solved':
           case.status = 'solved'

       db.session.commit()

       return jsonify({
           'status': 'success',
           'message': '反馈已收到'
       })
   ```

**验收标准**:
- 反馈数据正确保存
- 案例状态正确更新
- 支持解决方案修正

---

## 🎯 任务4: 知识文档管理API

### 4.1 文件上传API (第1天)

**目标**: 实现知识文档的上传功能。

**具体步骤**:

1. **文件上传处理** (`app/api/knowledge.py`)
   ```python
   import os
   from werkzeug.utils import secure_filename
   from flask import Blueprint, request, jsonify, current_app
   from flask_jwt_extended import jwt_required, get_jwt_identity

   bp = Blueprint('knowledge', __name__)

   ALLOWED_EXTENSIONS = {'pdf', 'doc', 'docx', 'txt', 'md', 'png', 'jpg', 'jpeg'}

   def allowed_file(filename):
       return '.' in filename and \
              filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

   @bp.route('/knowledge/documents', methods=['POST'])
   @jwt_required()
   def upload_document():
       user_id = get_jwt_identity()

       if 'file' not in request.files:
           return jsonify({'error': '没有选择文件'}), 400

       file = request.files['file']
       if file.filename == '':
           return jsonify({'error': '没有选择文件'}), 400

       if not allowed_file(file.filename):
           return jsonify({'error': '不支持的文件类型'}), 400

       # 获取表单数据
       tags = request.form.getlist('tags')
       vendor = request.form.get('vendor')

       # 保存文件
       filename = secure_filename(file.filename)
       file_id = str(uuid.uuid4())
       file_extension = filename.rsplit('.', 1)[1].lower()
       new_filename = f"{file_id}.{file_extension}"

       upload_folder = current_app.config['UPLOAD_FOLDER']
       os.makedirs(upload_folder, exist_ok=True)
       file_path = os.path.join(upload_folder, new_filename)
       file.save(file_path)

       # 保存文档记录
       document = KnowledgeDocument(
           id=file_id,
           filename=new_filename,
           original_filename=filename,
           file_path=file_path,
           file_size=os.path.getsize(file_path),
           mime_type=file.mimetype,
           vendor=vendor,
           tags=tags,
           user_id=user_id
       )

       db.session.add(document)
       db.session.flush()

       # 创建解析任务
       parsing_job = ParsingJob(
           document_id=document.id
       )
       db.session.add(parsing_job)
       db.session.commit()

       # 触发异步解析任务
       from app.services.document_service import parse_document
       parse_document.delay(parsing_job.id)

       return jsonify({
           'docId': document.id,
           'status': 'QUEUED',
           'message': '文档已加入处理队列'
       })
   ```

**验收标准**:
- 文件上传功能正常
- 支持多种文件格式
- 文件安全性检查完善

### 4.2 文档列表和详情API (第2天)

**目标**: 实现文档的查询和管理功能。

**具体步骤**:

1. **文档列表API**
   ```python
   @bp.route('/knowledge/documents', methods=['GET'])
   @jwt_required()
   def get_documents():
       user_id = get_jwt_identity()

       # 获取查询参数
       status = request.args.get('status')
       vendor = request.args.get('vendor')
       page = int(request.args.get('page', 1))
       page_size = int(request.args.get('pageSize', 10))

       query = KnowledgeDocument.query.filter_by(user_id=user_id)

       if status:
           query = query.filter_by(status=status)
       if vendor:
           query = query.filter_by(vendor=vendor)

       # 分页
       pagination = query.paginate(
           page=page,
           per_page=page_size,
           error_out=False
       )

       documents = pagination.items

       return jsonify({
           'items': [{
               'docId': doc.id,
               'fileName': doc.original_filename,
               'vendor': doc.vendor,
               'tags': doc.tags,
               'status': doc.status,
               'uploadedAt': doc.uploaded_at.isoformat() + 'Z'
           } for doc in documents],
           'total': pagination.total,
           'page': page,
           'pageSize': page_size
       })
   ```

2. **文档详情API**
   ```python
   @bp.route('/knowledge/documents/<doc_id>', methods=['GET'])
   @jwt_required()
   def get_document_detail(doc_id):
       user_id = get_jwt_identity()

       document = KnowledgeDocument.query.filter_by(
           id=doc_id,
           user_id=user_id
       ).first()

       if not document:
           return jsonify({'error': '文档不存在'}), 404

       # 获取解析任务信息
       parsing_job = ParsingJob.query.filter_by(
           document_id=doc_id
       ).order_by(ParsingJob.created_at.desc()).first()

       return jsonify({
           'docId': document.id,
           'fileName': document.original_filename,
           'vendor': document.vendor,
           'tags': document.tags,
           'status': document.status,
           'progress': document.progress,
           'message': parsing_job.error_message if parsing_job else None,
           'createdAt': document.uploaded_at.isoformat() + 'Z',
           'updatedAt': document.processed_at.isoformat() + 'Z' if document.processed_at else None
       })
   ```

**验收标准**:
- 文档列表查询正常
- 支持分页和过滤
- 文档详情信息完整

### 4.3 文档删除和重解析API (第3天)

**目标**: 实现文档的删除和重新解析功能。

**具体步骤**:

1. **删除文档API**
   ```python
   @bp.route('/knowledge/documents/<doc_id>', methods=['DELETE'])
   @jwt_required()
   def delete_document(doc_id):
       user_id = get_jwt_identity()

       document = KnowledgeDocument.query.filter_by(
           id=doc_id,
           user_id=user_id
       ).first()

       if not document:
           return jsonify({'error': '文档不存在'}), 404

       # 删除物理文件
       if os.path.exists(document.file_path):
           os.remove(document.file_path)

       # 删除向量数据库中的数据
       from app.services.vector_service import delete_document_vectors
       delete_document_vectors.delay(doc_id)

       # 删除数据库记录
       db.session.delete(document)
       db.session.commit()

       return '', 204
   ```

2. **重新解析API**
   ```python
   @bp.route('/knowledge/documents/<doc_id>/reparse', methods=['PUT'])
   @jwt_required()
   def reparse_document(doc_id):
       user_id = get_jwt_identity()

       document = KnowledgeDocument.query.filter_by(
           id=doc_id,
           user_id=user_id
       ).first()

       if not document:
           return jsonify({'error': '文档不存在'}), 404

       # 重置状态
       document.status = 'QUEUED'
       document.progress = 0
       document.error_message = None

       # 创建新的解析任务
       parsing_job = ParsingJob(
           document_id=document.id
       )
       db.session.add(parsing_job)
       db.session.commit()

       # 触发异步解析
       from app.services.document_service import parse_document
       parse_document.delay(parsing_job.id)

       return jsonify({
           'docId': document.id,
           'status': 'PARSING',
           'message': '已触发重新解析'
       })
   ```

**验收标准**:
- 文档删除功能正常
- 重新解析功能正常
- 相关数据清理完整

---

## 🎯 任务5: 异步任务处理系统

### 5.1 RQ任务队列配置 (第1天)

**目标**: 配置Redis和RQ任务队列系统。

**具体步骤**:

1. **Redis配置** (docker-compose.local.yml)
   ```yaml
   redis:
     image: redis:7-alpine
     ports:
       - "6379:6379"
     volumes:
       - redis_data:/data
   ```

2. **RQ配置** (`app/services/__init__.py`)
   ```python
   import redis
   from rq import Queue
   from flask import current_app

   def get_redis_connection():
       return redis.from_url(current_app.config['REDIS_URL'])

   def get_task_queue():
       return Queue('default', connection=get_redis_connection())
   ```

3. **Worker启动脚本** (`worker.py`)
   ```python
   import os
   from rq import Worker
   from app import create_app
   from app.services import get_redis_connection

   if __name__ == '__main__':
       app = create_app()
       with app.app_context():
           redis_conn = get_redis_connection()
           worker = Worker(['default'], connection=redis_conn)
           worker.work()
   ```

**验收标准**:
- Redis服务正常运行
- RQ队列配置正确
- Worker进程能正常启动

### 5.2 文档解析服务 (第2天)

**目标**: 实现异步文档解析服务。

**具体步骤**:

1. **文档解析任务** (`app/services/document_service.py`)
   ```python
   from rq import get_current_job
   from app import create_app, db
   from app.models.knowledge import KnowledgeDocument, ParsingJob
   from app.services.idp_service import IDPService
   from app.services.vector_service import VectorService

   def parse_document(job_id):
       app = create_app()
       with app.app_context():
           job = ParsingJob.query.get(job_id)
           if not job:
               return

           document = KnowledgeDocument.query.get(job.document_id)
           if not document:
               return

           try:
               # 更新状态
               job.status = 'PROCESSING'
               document.status = 'PARSING'
               db.session.commit()

               # 调用IDP服务解析文档
               idp_service = IDPService()
               parsed_result = idp_service.parse_document(document.file_path)

               # 语义切分
               from app.services.semantic_splitter import SemanticSplitter
               splitter = SemanticSplitter()
               chunks = splitter.split_document(parsed_result, document)

               # 向量化并存储
               vector_service = VectorService()
               vector_service.index_chunks(chunks, document.id)

               # 更新状态
               job.status = 'COMPLETED'
               job.result_data = {'chunks_count': len(chunks)}
               document.status = 'INDEXED'
               document.progress = 100
               document.processed_at = datetime.utcnow()

               db.session.commit()

           except Exception as e:
               # 错误处理
               job.status = 'FAILED'
               job.error_message = str(e)
               document.status = 'FAILED'
               document.error_message = str(e)

               db.session.commit()
               raise
   ```

2. **IDP服务封装** (`app/services/idp_service.py`)
   ```python
   from alibabacloud_docmind_api20220711.client import Client as docmind_api20220711Client
   from alibabacloud_tea_openapi import models as open_api_models
   from alibabacloud_docmind_api20220711 import models as docmind_api20220711_models
   from alibabacloud_tea_util import models as util_models
   from alibabacloud_credentials.client import Client as CredClient
   import os
   import time

   class IDPService:
       def __init__(self):
           # 使用默认凭证初始化Credentials Client
           cred = CredClient()
           config = open_api_models.Config(
               access_key_id=cred.get_credential().access_key_id,
               access_key_secret=cred.get_credential().access_key_secret
           )
           config.endpoint = 'docmind-api.cn-hangzhou.aliyuncs.com'
           self.client = docmind_api20220711Client(config)

       def parse_document(self, file_path):
           """调用阿里云文档智能API解析文档"""
           try:
               # 步骤1: 提交文档解析任务
               request = docmind_api20220711_models.SubmitDocStructureJobAdvanceRequest(
                   file_url_object=open(file_path, "rb"),
                   file_name=os.path.basename(file_path),
                   structure_type='default',  # 返回完整结构化信息
                   formula_enhancement=True   # 开启公式识别增强
               )
               runtime = util_models.RuntimeOptions()

               response = self.client.submit_doc_structure_job_advance(request, runtime)
               job_id = response.body.data.id

               # 步骤2: 轮询获取结果
               while True:
                   query_request = docmind_api20220711_models.GetDocStructureResultRequest(
                       id=job_id,
                       reveal_markdown=True  # 输出Markdown格式
                   )

                   query_response = self.client.get_doc_structure_result(query_request)

                   if query_response.body.completed:
                       if query_response.body.status == 'Success':
                           return query_response.body.data
                       else:
                           raise Exception(f"文档解析失败: {query_response.body.message}")

                   time.sleep(10)  # 建议每10秒轮询一次

           except Exception as e:
               raise Exception(f"IDP服务调用失败: {str(e)}")

       def parse_document_from_url(self, file_url, file_name):
           """通过URL解析文档"""
           try:
               request = docmind_api20220711_models.SubmitDocStructureJobRequest(
                   file_url=file_url,
                   file_name=file_name,
                   structure_type='default',
                   formula_enhancement=True
               )

               response = self.client.submit_doc_structure_job(request)
               job_id = response.body.data.id

               # 轮询获取结果 (同上逻辑)
               while True:
                   query_request = docmind_api20220711_models.GetDocStructureResultRequest(
                       id=job_id,
                       reveal_markdown=True
                   )

                   query_response = self.client.get_doc_structure_result(query_request)

                   if query_response.body.completed:
                       if query_response.body.status == 'Success':
                           return query_response.body.data
                       else:
                           raise Exception(f"文档解析失败: {query_response.body.message}")

                   time.sleep(10)

           except Exception as e:
               raise Exception(f"IDP服务调用失败: {str(e)}")
   ```

**验收标准**:
- 异步解析任务正常执行
- IDP服务集成成功
- 错误处理机制完善

### 5.3 Agent处理服务 (第3天)

**目标**: 实现Agent多轮对话的异步处理。

**具体步骤**:

1. **Agent服务** (`app/services/agent_service.py`)
   ```python
   from app.services.retrieval_service import RetrievalService
   from app.services.llm_service import LLMService
   from langchain_openai import ChatOpenAI
   from langchain.schema import HumanMessage, SystemMessage
   import os

   def analyze_user_query(case_id, node_id, query):
       app = create_app()
       with app.app_context():
           try:
               # 获取节点
               node = Node.query.get(node_id)
               if not node:
                   return

               # 问题分析
               llm_service = LLMService()
               analysis_result = llm_service.analyze_query(query)

               # 检索相关知识
               retrieval_service = RetrievalService()
               context = retrieval_service.search(query)

               # 判断是否需要更多信息
               if analysis_result.get('need_more_info'):
                   # 生成追问
                   clarification = llm_service.generate_clarification(
                       query, analysis_result, context
                   )

                   node.type = 'AI_CLARIFICATION'
                   node.title = '需要更多信息'
                   node.status = 'AWAITING_USER_INPUT'
                   node.content = {
                       'analysis': analysis_result.get('analysis'),
                       'suggestedQuestions': clarification.get('questions')
                   }
               else:
                   # 生成解决方案
                   solution = llm_service.generate_solution(
                       query, context, analysis_result
                   )

                   node.type = 'SOLUTION'
                   node.title = '解决方案'
                   node.status = 'COMPLETED'
                   node.content = {
                       'answer': solution.get('answer'),
                       'sources': solution.get('sources'),
                       'commands': solution.get('commands')
                   }

               db.session.commit()

           except Exception as e:
               # 错误处理
               node.status = 'COMPLETED'
               node.content = {
                   'error': '处理过程中发生错误，请重试'
               }
               db.session.commit()
               raise

   def process_user_response(case_id, node_id, response_data, retrieval_weight, filter_tags):
       # 处理用户补充信息的逻辑
       pass
   ```

**验收标准**:
- Agent异步处理正常
- 多轮对话逻辑正确
- 节点状态更新及时

### 5.4 任务监控和重试 (第4天)

**目标**: 实现任务监控和失败重试机制。

**具体步骤**:

1. **任务监控** (`app/services/task_monitor.py`)
   ```python
   from rq import get_current_job
   import logging

   def monitor_task_progress(func):
       def wrapper(*args, **kwargs):
           job = get_current_job()
           if job:
               job.meta['status'] = 'running'
               job.save_meta()

           try:
               result = func(*args, **kwargs)
               if job:
                   job.meta['status'] = 'completed'
                   job.save_meta()
               return result
           except Exception as e:
               if job:
                   job.meta['status'] = 'failed'
                   job.meta['error'] = str(e)
                   job.save_meta()
               logging.error(f"Task failed: {e}")
               raise
       return wrapper
   ```

2. **重试机制**
   ```python
   from rq import Retry

   # 在任务装饰器中添加重试
   @monitor_task_progress
   def parse_document(job_id):
       # 任务逻辑
       pass

   # 提交任务时指定重试策略
   queue.enqueue(
       parse_document,
       job_id,
       retry=Retry(max=3, interval=[10, 30, 60])
   )
   ```

**验收标准**:
- 任务监控功能正常
- 失败重试机制有效
- 日志记录完整

---

## 🎯 任务6: 数据看板API实现

### 6.1 统计数据聚合 (第1天)

**目标**: 实现数据看板所需的统计API。

**具体步骤**:

1. **统计API** (`app/api/statistics.py`)
   ```python
   from flask import Blueprint, request, jsonify
   from flask_jwt_extended import jwt_required
   from sqlalchemy import func, and_
   from datetime import datetime, timedelta

   bp = Blueprint('statistics', __name__)

   @bp.route('/statistics', methods=['GET'])
   @jwt_required()
   def get_statistics():
       time_range = request.args.get('timeRange', '30d')

       # 计算时间范围
       if time_range == '7d':
           start_date = datetime.utcnow() - timedelta(days=7)
       elif time_range == '90d':
           start_date = datetime.utcnow() - timedelta(days=90)
       else:  # 30d
           start_date = datetime.utcnow() - timedelta(days=30)

       # 故障分类统计
       fault_categories = db.session.query(
           Node.metadata['category'].astext.label('category'),
           func.count(Node.id).label('count')
       ).filter(
           and_(
               Node.type == 'USER_QUERY',
               Node.created_at >= start_date
           )
       ).group_by(Node.metadata['category'].astext).all()

       # 解决率趋势
       resolution_trend = []
       for i in range(7):
           date = datetime.utcnow() - timedelta(days=i)
           date_str = date.strftime('%Y-%m-%d')

           total_cases = Case.query.filter(
               func.date(Case.created_at) == date.date()
           ).count()

           solved_cases = Case.query.filter(
               and_(
                   func.date(Case.created_at) == date.date(),
                   Case.status == 'solved'
               )
           ).count()

           rate = solved_cases / total_cases if total_cases > 0 else 0
           resolution_trend.append({
               'date': date_str,
               'rate': round(rate, 2)
           })

       # 知识覆盖度
       knowledge_coverage = db.session.query(
           KnowledgeDocument.vendor,
           func.count(KnowledgeDocument.id).label('doc_count')
       ).filter(
           KnowledgeDocument.status == 'INDEXED'
       ).group_by(KnowledgeDocument.vendor).all()

       return jsonify({
           'faultCategories': [
               {'name': cat.category or 'Other', 'value': cat.count}
               for cat in fault_categories
           ],
           'resolutionTrend': resolution_trend,
           'knowledgeCoverage': [
               {
                   'topic': 'General',
                   'vendor': cov.vendor or 'Unknown',
                   'coverage': min(cov.doc_count * 10, 100)  # 简化计算
               }
               for cov in knowledge_coverage
           ]
       })
   ```

**验收标准**:
- 统计数据计算正确
- 支持不同时间范围
- 返回格式符合前端需求

### 6.2 性能优化 (第2天)

**目标**: 优化统计查询性能。

**具体步骤**:

1. **添加数据库索引**
   ```python
   # 在模型中添加索引
   class Node(db.Model):
       # ... 其他字段
       created_at = db.Column(db.DateTime, default=datetime.utcnow, index=True)
       type = db.Column(db.Enum(...), index=True)

   class Case(db.Model):
       # ... 其他字段
       created_at = db.Column(db.DateTime, default=datetime.utcnow, index=True)
       status = db.Column(db.Enum(...), index=True)
   ```

2. **缓存机制**
   ```python
   import redis
   import json
   from functools import wraps

   def cache_result(expire_time=300):
       def decorator(func):
           @wraps(func)
           def wrapper(*args, **kwargs):
               # 生成缓存key
               cache_key = f"stats:{func.__name__}:{hash(str(args) + str(kwargs))}"

               # 尝试从缓存获取
               redis_client = get_redis_connection()
               cached = redis_client.get(cache_key)
               if cached:
                   return json.loads(cached)

               # 执行函数并缓存结果
               result = func(*args, **kwargs)
               redis_client.setex(cache_key, expire_time, json.dumps(result))
               return result
           return wrapper
       return decorator

   @cache_result(expire_time=600)  # 缓存10分钟
   def get_statistics():
       # 统计逻辑
       pass
   ```

**验收标准**:
- 查询性能显著提升
- 缓存机制工作正常
- 数据一致性保证

---

## 🎯 任务7: 数据收集与IDP集成

### 7.1 数据收集准备 (第1天)

**目标**: 收集网络运维相关的知识文档，为知识库构建做准备。

**负责人**: 模型团队主导，后端团队协助

**具体步骤**:

1. **创建数据目录结构**
   ```bash
   mkdir -p assets/datasets/raw/{rfc,manuals,cases,others}
   mkdir -p assets/datasets/processed
   mkdir -p assets/datasets/embeddings
   ```

2. **收集RFC文档** (重点收集以下RFC)
   - RFC 2328 (OSPFv2)
   - RFC 4271 (BGP-4)
   - RFC 3031 (MPLS架构)
   - RFC 4364 (BGP/MPLS IP VPN)
   - RFC 2547 (BGP/MPLS VPN)

3. **收集设备手册**
   - 华为: VRP系统配置指南、故障处理手册
   - 思科: IOS配置指南、故障排除手册
   - 华三: Comware系统手册

4. **收集案例文档**
   - 网络故障案例分析
   - 运维最佳实践文档
   - 技术博客和论坛精华帖

**验收标准**:
- 收集到至少50个高质量文档
- 文档按厂商和技术分类存放
- 建立文档清单Excel表格，记录文件名、来源、分类、厂商标签

### 7.2 阿里云文档智能服务集成 (第2天)

**目标**: 集成阿里云文档智能服务到后端异步任务系统中。

**负责人**: 模型团队主导，后端团队配合

**具体步骤**:

1. **完善IDP服务封装** (`app/services/idp_service.py`)
   ```python
   from alibabacloud_docmind_api20220711.client import Client as docmind_api20220711Client
   from alibabacloud_tea_openapi import models as open_api_models
   from alibabacloud_docmind_api20220711 import models as docmind_api20220711_models
   from alibabacloud_tea_util import models as util_models
   from alibabacloud_credentials.client import Client as CredClient
   import os
   import time

   class IDPService:
       def __init__(self):
           # 使用默认凭证初始化Credentials Client
           cred = CredClient()
           config = open_api_models.Config(
               access_key_id=cred.get_credential().access_key_id,
               access_key_secret=cred.get_credential().access_key_secret
           )
           config.endpoint = 'docmind-api.cn-hangzhou.aliyuncs.com'
           self.client = docmind_api20220711Client(config)

       def parse_document(self, file_path):
           """调用阿里云文档智能API解析文档"""
           try:
               # 步骤1: 提交文档解析任务
               request = docmind_api20220711_models.SubmitDocStructureJobAdvanceRequest(
                   file_url_object=open(file_path, "rb"),
                   file_name=os.path.basename(file_path),
                   structure_type='default',  # 返回完整结构化信息
                   formula_enhancement=True   # 开启公式识别增强
               )
               runtime = util_models.RuntimeOptions()

               response = self.client.submit_doc_structure_job_advance(request, runtime)
               job_id = response.body.data.id

               # 步骤2: 轮询获取结果
               while True:
                   query_request = docmind_api20220711_models.GetDocStructureResultRequest(
                       id=job_id,
                       reveal_markdown=True  # 输出Markdown格式
                   )

                   query_response = self.client.get_doc_structure_result(query_request)

                   if query_response.body.completed:
                       if query_response.body.status == 'Success':
                           return query_response.body.data
                       else:
                           raise Exception(f"文档解析失败: {query_response.body.message}")

                   time.sleep(10)  # 建议每10秒轮询一次

           except Exception as e:
               raise Exception(f"IDP服务调用失败: {str(e)}")
   ```

2. **实现语义切分器** (`app/services/semantic_splitter.py`)
   ```python
   class SemanticSplitter:
       def __init__(self, max_chunk_size=1000, overlap=100):
           self.max_chunk_size = max_chunk_size
           self.overlap = overlap

       def split_document(self, idp_result, document):
           """基于IDP结果进行语义切分"""
           chunks = []

           # 解析新的IDP返回结构
           layouts = idp_result.get('layouts', [])
           doc_tree = idp_result.get('logics', {}).get('docTree', [])

           # 按照文档层级树进行语义切分
           for layout in layouts:
               chunk = {
                   'content': layout.get('text', ''),
                   'title': self._extract_title(layout),
                   'type': layout.get('type', 'text'),
                   'subtype': layout.get('subType', ''),
                   'page_number': layout.get('pageNum', [0])[0] if layout.get('pageNum') else 0,
                   'unique_id': layout.get('uniqueId', ''),
                   'markdown_content': layout.get('markdownContent', '')  # 新增Markdown内容
               }

               # 根据内容长度决定是否需要进一步切分
               if len(chunk['content']) > self.max_chunk_size:
                   sub_chunks = self._split_large_content(chunk)
                   chunks.extend(sub_chunks)
               else:
                   chunks.append(chunk)

           return chunks

       def _extract_title(self, layout):
           """从layout中提取标题"""
           if layout.get('type') in ['title', 'para_title']:
               return layout.get('text', '')
           return ''

       def _split_large_content(self, chunk):
           """切分过长的内容"""
           content = chunk['content']
           sub_chunks = []

           for i in range(0, len(content), self.max_chunk_size - self.overlap):
               sub_content = content[i:i + self.max_chunk_size]
               sub_chunk = chunk.copy()
               sub_chunk['content'] = sub_content
               sub_chunks.append(sub_chunk)

           return sub_chunks

       def extract_metadata(self, chunk, document):
           """提取文档片段的元数据"""
           return {
               'vendor': self._detect_vendor(chunk['content']),
               'category': self._classify_content(chunk['content']),
               'source_document': document.original_filename,
               'page_number': chunk.get('page_number', 0),
               'element_type': chunk.get('type', 'text'),
               'element_subtype': chunk.get('subtype', ''),
               'unique_id': chunk.get('unique_id', ''),
               'has_markdown': bool(chunk.get('markdown_content'))
           }

       def _detect_vendor(self, content):
           """检测厂商信息"""
           content_lower = content.lower()
           if any(keyword in content_lower for keyword in ['华为', 'huawei', 'vrp']):
               return '华为'
           elif any(keyword in content_lower for keyword in ['思科', 'cisco', 'ios']):
               return '思科'
           elif any(keyword in content_lower for keyword in ['华三', 'h3c', 'comware']):
               return '华三'
           return '通用'

       def _classify_content(self, content):
           """分类内容"""
           content_lower = content.lower()
           if any(keyword in content_lower for keyword in ['ospf', 'open shortest path first']):
               return 'OSPF'
           elif any(keyword in content_lower for keyword in ['bgp', 'border gateway protocol']):
               return 'BGP'
           elif any(keyword in content_lower for keyword in ['mpls', 'multiprotocol label switching']):
               return 'MPLS'
           elif any(keyword in content_lower for keyword in ['vpn', 'virtual private network']):
               return 'VPN'
           return '其他'
   ```

**验收标准**:
- IDP服务集成到后端异步任务系统
- 语义切分算法实现完成
- 能处理PDF、Word、图片等多种格式

### 7.3 测试IDP集成 (第3天)

**目标**: 测试文档解析和切分功能。

**具体步骤**:

1. **编写测试用例**
   ```python
   def test_idp_integration():
       # 测试不同类型文档解析
       # 测试语义切分效果
       # 测试元数据提取
       pass
   ```

2. **性能测试**
   - 测试解析速度
   - 测试并发处理能力
   - 测试错误处理机制

**验收标准**:
- 所有测试用例通过
- 解析准确率>90%
- 平均解析时间<30秒/文档

---

## 🎯 任务8: 向量数据库构建与配置

### 8.1 Weaviate本地部署 (第1天)

**目标**: 使用Docker在本地部署Weaviate向量数据库。

**负责人**: 模型团队主导，后端团队协助Docker配置

**具体步骤**:

1. **更新Docker Compose配置** (`docker-compose.local.yml`)
   ```yaml
   version: '3.8'
   services:
     weaviate:
       image: semitechnologies/weaviate:1.21.2
       ports:
         - "8080:8080"
         - "50051:50051"
       environment:
         QUERY_DEFAULTS_LIMIT: 25
         AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'true'
         PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
         DEFAULT_VECTORIZER_MODULE: 'none'
         ENABLE_MODULES: 'text2vec-openai,generative-openai'
         CLUSTER_HOSTNAME: 'node1'
       volumes:
         - weaviate_data:/var/lib/weaviate

     mysql:
       image: mysql:8.0
       ports:
         - "3306:3306"
       environment:
         MYSQL_ROOT_PASSWORD: password
         MYSQL_DATABASE: ip_expert
       volumes:
         - mysql_data:/var/lib/mysql

     redis:
       image: redis:7-alpine
       ports:
         - "6379:6379"
       volumes:
         - redis_data:/data

   volumes:
     weaviate_data:
     mysql_data:
     redis_data:
   ```

2. **创建向量服务** (`app/services/vector_service.py`)
   ```python
   import weaviate
   from langchain_community.embeddings import DashScopeEmbeddings

   class VectorService:
       def __init__(self):
           self.client = weaviate.Client("http://localhost:8080")
           # 使用Langchain集成的DashScope向量模型
           self.embedding_service = DashScopeEmbeddings(
               model="text-embedding-v4"
           )

       def create_schema(self):
           schema = {
               "class": "KnowledgeChunk",
               "description": "网络运维知识片段",
               "properties": [
                   {"name": "content", "dataType": ["text"]},
                   {"name": "title", "dataType": ["string"]},
                   {"name": "vendor", "dataType": ["string"]},
                   {"name": "category", "dataType": ["string"]},
                   {"name": "source_document", "dataType": ["string"]},
                   {"name": "page_number", "dataType": ["int"]}
               ],
               "vectorizer": "none"
           }
           self.client.schema.create_class(schema)

       def index_chunks(self, chunks, document_id):
           """批量索引文档片段"""
           with self.client.batch as batch:
               for chunk in chunks:
                   vector = self.embedding_service.embed_text(chunk['content'])

                   batch.add_data_object(
                       data_object=chunk,
                       class_name="KnowledgeChunk",
                       vector=vector
                   )
   ```

**验收标准**:
- Weaviate服务正常启动
- Schema创建成功
- 支持向量索引和查询

### 8.2 向量化模型集成 (第2天)

**目标**: 集成阿里云百炼平台的text-embedding-v4模型。

**负责人**: 模型团队

**具体步骤**:

1. **创建嵌入服务** (`app/services/embedding_service.py`)
   ```python
   from langchain_community.embeddings import DashScopeEmbeddings
   from typing import List
   import os

   class QwenEmbedding:
       def __init__(self, api_key=None):
           """使用Langchain集成的DashScope向量模型"""
           self.api_key = api_key or os.environ.get('DASHSCOPE_API_KEY')
           self.embeddings = DashScopeEmbeddings(
               model="text-embedding-v4",
               dashscope_api_key=self.api_key
           )

       def embed_text(self, text: str) -> List[float]:
           """单个文本向量化"""
           try:
               return self.embeddings.embed_query(text)
           except Exception as e:
               raise Exception(f"向量化失败: {str(e)}")

       def embed_batch(self, texts: List[str]) -> List[List[float]]:
           """批量文本向量化"""
           try:
               return self.embeddings.embed_documents(texts)
           except Exception as e:
               # 降级到单个处理
               embeddings = []
               for text in texts:
                   try:
                       emb = self.embed_text(text)
                       embeddings.append(emb)
                   except:
                       embeddings.append([0.0] * 1536)  # 默认维度
               return embeddings
   ```

**验收标准**:
- 能成功调用text-embedding-v4 API
- 支持批量向量化处理
- 向量维度正确(1536维)

---

## 🎯 任务9: 混合检索算法实现

### 9.1 两阶段检索实现 (第1-2天)

**目标**: 实现召回+精排的两阶段检索算法。

**负责人**: 模型团队主导，后端团队配合API集成

**具体步骤**:

1. **创建检索服务** (`app/services/retrieval_service.py`)
   ```python
   import weaviate
   from langchain_community.embeddings import DashScopeEmbeddings
   from typing import List, Dict

   class RetrievalService:
       def __init__(self):
           self.client = weaviate.Client("http://localhost:8080")
           # 使用Langchain集成的DashScope向量模型
           self.embedding_service = DashScopeEmbeddings(
               model="text-embedding-v4"
           )

       def hybrid_search(self, query: str, top_k: int = 5,
                        vendor_filter: str = None) -> List[Dict]:
           """混合检索：召回+精排"""

           # 第一阶段：召回
           candidates = self._recall_stage(query, vendor_filter)

           # 第二阶段：精排
           ranked_results = self._rerank_stage(query, candidates, top_k)

           return ranked_results

       def _recall_stage(self, query: str, vendor_filter: str = None) -> List[Dict]:
           """召回阶段：BM25 + 向量检索"""

           # BM25关键词检索
           bm25_results = self._bm25_search(query, vendor_filter, limit=50)

           # 向量语义检索
           vector_results = self._vector_search(query, vendor_filter, limit=50)

           # 合并去重
           combined_results = self._merge_results(bm25_results, vector_results)

           return combined_results

       def _bm25_search(self, query: str, vendor_filter: str = None, limit: int = 50):
           """BM25关键词检索"""
           where_filter = None
           if vendor_filter:
               where_filter = {"path": ["vendor"], "operator": "Equal", "valueString": vendor_filter}

           result = self.client.query.get("KnowledgeChunk", [
               "content", "title", "vendor", "category", "source_document", "page_number"
           ]).with_bm25(
               query=query
           ).with_where(where_filter).with_limit(limit).do()

           return result.get('data', {}).get('Get', {}).get('KnowledgeChunk', [])

       def _vector_search(self, query: str, vendor_filter: str = None, limit: int = 50):
           """向量语义检索"""
           query_vector = self.embedding_service.embed_text(query)

           where_filter = None
           if vendor_filter:
               where_filter = {"path": ["vendor"], "operator": "Equal", "valueString": vendor_filter}

           result = self.client.query.get("KnowledgeChunk", [
               "content", "title", "vendor", "category", "source_document", "page_number"
           ]).with_near_vector({
               "vector": query_vector
           }).with_where(where_filter).with_limit(limit).do()

           return result.get('data', {}).get('Get', {}).get('KnowledgeChunk', [])

       def _rerank_stage(self, query: str, candidates: List[Dict], top_k: int) -> List[Dict]:
           """精排阶段：使用Qwen3-Reranker"""
           if not candidates:
               return []

           # 调用重排序模型
           scores = self._call_reranker(query, candidates)

           # 按分数排序
           scored_candidates = list(zip(candidates, scores))
           scored_candidates.sort(key=lambda x: x[1], reverse=True)

           # 返回top_k结果
           return [candidate for candidate, score in scored_candidates[:top_k]]

       def _call_reranker(self, query: str, candidates: List[Dict]) -> List[float]:
           """调用OLLAMA部署的Qwen3-Reranker模型"""
           # TODO: 实现OLLAMA API调用
           # 暂时返回模拟分数
           import random
           return [random.random() for _ in candidates]
   ```

**验收标准**:
- 混合检索功能正常
- 召回率>80%
- 检索速度<500ms

### 9.2 重排序模型部署 (第3天)

**目标**: 使用OLLAMA部署Qwen3-Reranker模型。

**负责人**: 模型团队

**具体步骤**:

1. **安装OLLAMA**
   ```bash
   # macOS
   brew install ollama

   # Linux
   curl -fsSL https://ollama.ai/install.sh | sh
   ```

2. **部署重排序模型**
   ```bash
   ollama pull qwen3-reranker
   ollama serve
   ```

3. **集成重排序API** (`app/services/reranker_service.py`)
   ```python
   from langchain_community.document_compressors import LLMChainExtractor
   from langchain_community.llms import Ollama
   from langchain.schema import Document
   from typing import List, Dict
   import os

   class RerankerService:
       def __init__(self, ollama_url=None):
           """使用Langchain集成OLLAMA重排序模型"""
           self.ollama_url = ollama_url or os.environ.get('OLLAMA_BASE_URL', 'http://localhost:11434')

           # 使用Langchain的Ollama集成
           self.llm = Ollama(
               model="qwen3-reranker",
               base_url=self.ollama_url
           )

       def rerank(self, query: str, documents: List[str]) -> List[float]:
           """使用Qwen3-Reranker对文档重排序"""
           scores = []

           for doc in documents:
               # 构建重排序提示词
               prompt = f"""请评估以下文档与查询的相关性，返回0-1之间的分数：

查询: {query}

文档: {doc}

相关性分数:"""

               try:
                   response = self.llm.invoke(prompt)
                   score = self._parse_score(response)
                   scores.append(score)
               except Exception as e:
                   print(f"重排序失败: {e}")
                   scores.append(0.5)  # 默认分数

           return scores

       def _parse_score(self, response: str) -> float:
           """解析模型返回的相关性分数"""
           try:
               import re
               # 查找0-1之间的小数
               match = re.search(r'0\.\d+|1\.0|0|1', response)
               if match:
                   score = float(match.group())
                   return max(0.0, min(1.0, score))  # 确保在0-1范围内
           except:
               pass
           return 0.5  # 默认分数
   ```

**验收标准**:
- OLLAMA服务正常运行
- 重排序模型部署成功
- API调用正常

---

## 🎯 任务10: 提示词工程与优化

### 10.1 基础Prompt设计 (第1-2天)

**目标**: 设计用于不同场景的基础提示词模板。

**负责人**: 模型团队主导

**具体步骤**:

1. **创建Prompt模板库** (`app/prompts/`)
   ```python
   # app/prompts/analysis_prompt.py
   ANALYSIS_PROMPT = """
   你是一位资深的网络运维专家。请分析用户提出的网络问题。

   用户问题: {user_query}

   上下文信息:
   {context}

   请按以下格式分析:
   1. 问题分类: [OSPF/BGP/MPLS/VPN/其他]
   2. 可能原因: 列出3-5个可能的原因
   3. 需要补充的信息: 如果信息不足，列出需要用户提供的具体信息
   4. 初步建议: 给出初步的排查方向

   如果信息不足以给出完整分析，请在回答末尾添加: [NEED_MORE_INFO]
   """

   CLARIFICATION_PROMPT = """
   基于当前的分析，我需要更多信息来帮助您解决问题。

   当前分析: {current_analysis}

   请回答以下问题:
   {questions}

   您也可以上传相关的配置文件、日志文件或网络拓扑图。
   """

   SOLUTION_PROMPT = """
   你是一位{vendor}网络设备专家。基于以下信息，提供详细的解决方案。

   问题描述: {problem}
   相关文档: {retrieved_docs}
   用户补充信息: {user_context}

   请提供:
   1. 问题根因分析
   2. 详细解决步骤 (每步都要具体可执行)
   3. 验证方法
   4. 预防措施

   在引用文档内容时，请使用 [doc1], [doc2] 等标记。
   所有命令都要使用{vendor}的语法格式。
   """
   ```

2. **厂商适配Prompt**
   ```python
   # app/prompts/vendor_prompts.py
   HUAWEI_PROMPT = """
   你是华为VRP系统专家。请使用华为设备的命令语法回答问题。

   常用命令格式:
   - 查看接口: display interface
   - 查看路由: display ip routing-table
   - 查看OSPF: display ospf peer
   - 配置接口: interface GigabitEthernet0/0/1

   {base_prompt}

   注意: 所有命令必须符合VRP语法规范。
   """

   CISCO_PROMPT = """
   你是思科IOS系统专家。请使用思科设备的命令语法回答问题。

   常用命令格式:
   - 查看接口: show interfaces
   - 查看路由: show ip route
   - 查看OSPF: show ip ospf neighbor
   - 配置接口: interface GigabitEthernet0/0

   {base_prompt}

   注意: 所有命令必须符合IOS语法规范。
   """
   ```

**验收标准**:
- Prompt模板覆盖所有节点类型
- 支持厂商适配
- 输出格式规范统一

### 10.2 LLM服务集成 (第3天)

**目标**: 集成阿里云百炼平台的qwen-plus模型。

**负责人**: 模型团队主导，后端团队配合

**具体步骤**:

1. **创建LLM服务** (`app/services/llm_service.py`)
   ```python
   from langchain_openai import ChatOpenAI
   from langchain.schema import HumanMessage, SystemMessage
   from app.prompts.analysis_prompt import ANALYSIS_PROMPT, SOLUTION_PROMPT
   import os

   class LLMService:
       def __init__(self):
           self.llm = ChatOpenAI(
               model="qwen-plus",
               openai_api_key=os.environ.get('DASHSCOPE_API_KEY'),
               openai_api_base="https://dashscope.aliyuncs.com/compatible-mode/v1",
               temperature=0.1
           )

       def analyze_query(self, query: str, context: str = "") -> dict:
           """分析用户查询"""
           prompt = ANALYSIS_PROMPT.format(
               user_query=query,
               context=context
           )

           messages = [
               SystemMessage(content="你是一位资深的网络运维专家。"),
               HumanMessage(content=prompt)
           ]

           response = self.llm.invoke(messages)

           # 解析响应
           need_more_info = "[NEED_MORE_INFO]" in response.content

           return {
               "analysis": response.content,
               "need_more_info": need_more_info,
               "category": self._extract_category(response.content)
           }

       def generate_solution(self, query: str, context: list,
                           vendor: str = "通用") -> dict:
           """生成解决方案"""
           retrieved_docs = self._format_context(context)

           prompt = SOLUTION_PROMPT.format(
               vendor=vendor,
               problem=query,
               retrieved_docs=retrieved_docs,
               user_context=""
           )

           messages = [
               SystemMessage(content=f"你是一位{vendor}网络设备专家。"),
               HumanMessage(content=prompt)
           ]

           response = self.llm.invoke(messages)

           return {
               "answer": response.content,
               "sources": self._extract_sources(context),
               "commands": self._extract_commands(response.content)
           }

       def _extract_category(self, text: str) -> str:
           """从分析结果中提取问题分类"""
           # 简单的关键词匹配
           if "OSPF" in text:
               return "OSPF"
           elif "BGP" in text:
               return "BGP"
           elif "MPLS" in text:
               return "MPLS"
           else:
               return "其他"

       def _format_context(self, context: list) -> str:
           """格式化检索到的上下文"""
           formatted = ""
           for i, doc in enumerate(context, 1):
               formatted += f"[doc{i}] {doc.get('title', '')}\n{doc.get('content', '')}\n\n"
           return formatted

       def _extract_sources(self, context: list) -> list:
           """提取引用来源"""
           sources = []
           for i, doc in enumerate(context, 1):
               sources.append({
                   "id": f"doc{i}",
                   "file": doc.get('source_document', ''),
                   "page": doc.get('page_number', 0)
               })
           return sources

       def _extract_commands(self, text: str) -> list:
           """从回答中提取命令"""
           import re
           # 简单的命令提取逻辑
           commands = re.findall(r'`([^`]+)`', text)
           return commands
   ```

**验收标准**:
- LLM服务集成成功
- 支持流式和非流式输出
- 错误处理完善

---

## 🎯 任务11: langgraph Agent流程构建

### 11.1 状态机设计 (第1-2天)

**目标**: 设计多轮对话的状态机架构。

**负责人**: 模型团队主导，后端团队配合集成

**具体步骤**:

1. **定义Agent状态** (`app/services/agent_state.py`)
   ```python
   from typing import TypedDict, List, Dict, Optional

   class AgentState(TypedDict):
       messages: List[Dict]      # 对话历史
       context: List[Dict]       # 检索到的知识
       user_query: str          # 用户问题
       vendor: Optional[str]    # 设备厂商
       category: Optional[str]  # 问题分类
       need_more_info: bool     # 是否需要更多信息
       solution_ready: bool     # 是否可以给出解决方案
       case_id: str            # 案例ID
       current_node_id: str    # 当前节点ID
   ```

2. **实现Agent节点** (`app/services/agent_nodes.py`)
   ```python
   from app.services.llm_service import LLMService
   from app.services.retrieval_service import RetrievalService
   from app.models.case import Node
   from app import db

   def analyze_query(state: AgentState) -> AgentState:
       """分析用户问题"""
       llm_service = LLMService()

       analysis_result = llm_service.analyze_query(
           state["user_query"],
           context=""
       )

       # 更新状态
       state["category"] = analysis_result.get("category")
       state["need_more_info"] = analysis_result.get("need_more_info", False)

       # 更新数据库节点
       node = Node.query.get(state["current_node_id"])
       if node:
           node.content = {
               "analysis": analysis_result.get("analysis"),
               "category": analysis_result.get("category")
           }
           if state["need_more_info"]:
               node.type = "AI_CLARIFICATION"
               node.status = "AWAITING_USER_INPUT"
           else:
               node.type = "AI_ANALYSIS"
               node.status = "COMPLETED"
           db.session.commit()

       return state

   def retrieve_knowledge(state: AgentState) -> AgentState:
       """检索相关知识"""
       retrieval_service = RetrievalService()

       context = retrieval_service.hybrid_search(
           query=state["user_query"],
           vendor_filter=state.get("vendor")
       )

       state["context"] = context
       return state

   def generate_solution(state: AgentState) -> AgentState:
       """生成解决方案"""
       llm_service = LLMService()

       solution = llm_service.generate_solution(
           query=state["user_query"],
           context=state["context"],
           vendor=state.get("vendor", "通用")
       )

       # 更新数据库节点
       node = Node.query.get(state["current_node_id"])
       if node:
           node.type = "SOLUTION"
           node.title = "解决方案"
           node.status = "COMPLETED"
           node.content = {
               "answer": solution.get("answer"),
               "sources": solution.get("sources"),
               "commands": solution.get("commands")
           }
           db.session.commit()

       state["solution_ready"] = True
       return state
   ```

**验收标准**:
- 状态结构设计合理
- 节点功能实现完整
- 与数据库集成正常

### 11.2 工作流编排 (第3-4天)

**目标**: 使用langgraph构建完整的对话流程。

**负责人**: 模型团队主导

**具体步骤**:

1. **创建Agent工作流** (`app/services/agent_workflow.py`)
   ```python
   from langgraph.graph import StateGraph, END
   from app.services.agent_state import AgentState
   from app.services.agent_nodes import analyze_query, retrieve_knowledge, generate_solution

   def create_agent_workflow():
       """创建Agent工作流"""
       workflow = StateGraph(AgentState)

       # 添加节点
       workflow.add_node("analyze", analyze_query)
       workflow.add_node("retrieve", retrieve_knowledge)
       workflow.add_node("solve", generate_solution)

       # 设置入口点
       workflow.set_entry_point("analyze")

       # 定义条件边
       workflow.add_conditional_edges(
           "analyze",
           should_retrieve_or_end,
           {
               "retrieve": "retrieve",
               "end": END
           }
       )

       workflow.add_edge("retrieve", "solve")
       workflow.add_edge("solve", END)

       return workflow.compile()

   def should_retrieve_or_end(state: AgentState) -> str:
       """判断是否需要检索知识"""
       if state.get("need_more_info"):
           return "end"  # 需要更多信息，等待用户输入
       return "retrieve"
   ```

2. **集成到异步任务** (更新 `app/services/agent_service.py`)
   ```python
   from app.services.agent_workflow import create_agent_workflow

   def analyze_user_query(case_id, node_id, query):
       """分析用户查询的异步任务"""
       app = create_app()
       with app.app_context():
           try:
               # 创建Agent工作流
               agent = create_agent_workflow()

               # 初始化状态
               initial_state = {
                   "user_query": query,
                   "messages": [],
                   "context": [],
                   "need_more_info": False,
                   "solution_ready": False,
                   "case_id": case_id,
                   "current_node_id": node_id
               }

               # 执行工作流
               final_state = agent.invoke(initial_state)

           except Exception as e:
               # 错误处理
               node = Node.query.get(node_id)
               if node:
                   node.status = "COMPLETED"
                   node.content = {"error": "处理过程中发生错误，请重试"}
                   db.session.commit()
               raise
   ```

**验收标准**:
- 工作流逻辑正确
- 与后端异步任务集成
- 支持多轮对话

---

## 🎯 任务12: 模型接口集成与测试

### 12.1 性能优化 (第1天)

**目标**: 优化模型调用性能，添加监控。

**负责人**: 模型团队

**具体步骤**:

1. **添加缓存机制**
   ```python
   # app/services/cache_service.py
   import redis
   import hashlib
   import json

   class CacheService:
       def __init__(self):
           self.redis_client = redis.from_url(os.environ.get('REDIS_URL'))

       def get_cached_result(self, key: str):
           """获取缓存结果"""
           cached = self.redis_client.get(key)
           if cached:
               return json.loads(cached)
           return None

       def cache_result(self, key: str, result: dict, expire_time: int = 3600):
           """缓存结果"""
           self.redis_client.setex(key, expire_time, json.dumps(result))

       def generate_cache_key(self, prefix: str, *args) -> str:
           """生成缓存键"""
           content = f"{prefix}:{':'.join(map(str, args))}"
           return hashlib.md5(content.encode()).hexdigest()
   ```

2. **添加性能监控**
   ```python
   # app/utils/monitoring.py
   import time
   import logging
   from functools import wraps

   def monitor_performance(operation_name: str):
       def decorator(func):
           @wraps(func)
           def wrapper(*args, **kwargs):
               start_time = time.time()
               try:
                   result = func(*args, **kwargs)
                   duration = time.time() - start_time
                   logging.info(f"{operation_name} success: {duration:.2f}s")
                   return result
               except Exception as e:
                   duration = time.time() - start_time
                   logging.error(f"{operation_name} failed: {duration:.2f}s, {e}")
                   raise
           return wrapper
       return decorator
   ```

**验收标准**:
- 缓存机制工作正常
- 性能监控数据准确
- 平均响应时间<3秒

### 12.2 集成测试 (第2天)

**目标**: 测试所有AI组件的集成效果。

**负责人**: 全团队

**具体步骤**:

1. **端到端测试**
   ```python
   def test_full_ai_pipeline():
       """测试完整的AI流程"""
       # 测试文档上传和解析
       # 测试向量化和索引
       # 测试检索和重排序
       # 测试Agent对话流程
       pass
   ```

**验收标准**:
- 所有AI组件集成测试通过
- 端到端流程正常
- 性能指标达标

---

## 🎯 任务13: 系统集成与端到端测试

### 13.1 API集成测试 (第1-2天)

**目标**: 进行完整的API集成测试。

**负责人**: 全团队协作

**具体步骤**:

1. **测试用例编写** (`tests/test_api.py`)
   ```python
   import pytest
   from app import create_app, db
   from app.models.user import User

   @pytest.fixture
   def app():
       app = create_app('testing')
       with app.app_context():
           db.create_all()
           yield app
           db.drop_all()

   @pytest.fixture
   def client(app):
       return app.test_client()

   @pytest.fixture
   def auth_headers(client):
       # 创建测试用户并获取token
       user = User(username='test', email='test@example.com')
       user.set_password('password')
       db.session.add(user)
       db.session.commit()

       response = client.post('/api/v1/auth/login', json={
           'username': 'test',
           'password': 'password'
       })
       token = response.get_json()['access_token']
       return {'Authorization': f'Bearer {token}'}

   def test_create_case(client, auth_headers):
       response = client.post('/api/v1/cases',
           headers=auth_headers,
           json={
               'query': 'OSPF邻居建立失败',
               'attachments': []
           }
       )
       assert response.status_code == 200
       data = response.get_json()
       assert 'caseId' in data
       assert len(data['nodes']) == 2
   ```

2. **自动化测试脚本**
   ```bash
   #!/bin/bash
   # run_tests.sh

   echo "Running API tests..."
   pytest tests/test_api.py -v

   echo "Running integration tests..."
   pytest tests/test_integration.py -v

   echo "Running performance tests..."
   pytest tests/test_performance.py -v
   ```

**验收标准**:
- 所有API测试通过
- 集成测试覆盖主要流程
- 性能测试满足要求

### 13.2 AI流程集成测试 (第2天)

**目标**: 测试完整的AI处理流程。

**负责人**: 模型团队主导，后端团队配合

**具体步骤**:

1. **文档处理流程测试**
   ```python
   def test_document_processing_pipeline():
       """测试文档处理完整流程"""
       # 上传文档 -> IDP解析 -> 语义切分 -> 向量化 -> 索引
       # 验证每个步骤的输出
       pass

   def test_knowledge_retrieval():
       """测试知识检索流程"""
       # 用户查询 -> 混合检索 -> 重排序 -> 返回结果
       # 验证检索质量和性能
       pass

   def test_agent_conversation():
       """测试Agent对话流程"""
       # 用户问题 -> 分析 -> 检索 -> 生成方案
       # 验证多轮对话逻辑
       pass
   ```

2. **性能基准测试**
   ```python
   def test_performance_benchmarks():
       """性能基准测试"""
       # 文档解析速度: <30秒/文档
       # 检索响应时间: <500ms
       # LLM生成时间: <3秒
       # 并发处理能力: 10个并发请求
       pass
   ```

**验收标准**:
- 所有AI流程测试通过
- 性能指标达到要求
- 错误处理机制完善

### 13.3 错误处理和日志 (第3天)

**目标**: 完善错误处理和日志系统。

**负责人**: 后端团队主导

**具体步骤**:

1. **全局错误处理** (`app/errors.py`)
   ```python
   from flask import jsonify
   from app import db

   def register_error_handlers(app):
       @app.errorhandler(400)
       def bad_request(error):
           return jsonify({
               'code': 400,
               'status': 'error',
               'error': {
                   'type': 'INVALID_REQUEST',
                   'message': '请求参数错误'
               }
           }), 400

       @app.errorhandler(401)
       def unauthorized(error):
           return jsonify({
               'code': 401,
               'status': 'error',
               'error': {
                   'type': 'UNAUTHORIZED',
                   'message': '未授权访问'
               }
           }), 401

       @app.errorhandler(500)
       def internal_error(error):
           db.session.rollback()
           return jsonify({
               'code': 500,
               'status': 'error',
               'error': {
                   'type': 'INTERNAL_ERROR',
                   'message': '服务器内部错误'
               }
           }), 500
   ```

2. **日志配置** (`app/logging_config.py`)
   ```python
   import logging
   from logging.handlers import RotatingFileHandler
   import os

   def setup_logging(app):
       if not app.debug:
           if not os.path.exists('logs'):
               os.mkdir('logs')

           file_handler = RotatingFileHandler(
               'logs/ip_expert.log',
               maxBytes=10240000,
               backupCount=10
           )
           file_handler.setFormatter(logging.Formatter(
               '%(asctime)s %(levelname)s: %(message)s [in %(pathname)s:%(lineno)d]'
           ))
           file_handler.setLevel(logging.INFO)
           app.logger.addHandler(file_handler)

           app.logger.setLevel(logging.INFO)
           app.logger.info('IP Expert startup')
   ```

**验收标准**:
- 错误处理覆盖所有场景
- 日志记录详细准确
- 错误响应格式统一

### 13.4 部署准备 (第4天)

**目标**: 准备生产环境部署配置。

**负责人**: 后端团队主导，模型团队配合

**具体步骤**:

1. **Docker化** (`Dockerfile`)
   ```dockerfile
   FROM python:3.9-slim

   WORKDIR /app

   COPY requirements.txt .
   RUN pip install -r requirements.txt

   COPY . .

   EXPOSE 5000

   CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:create_app()"]
   ```

2. **生产配置** (`docker-compose.prod.yml`)
   ```yaml
   version: '3.8'
   services:
     backend:
       build: .
       ports:
         - "5000:5000"
       environment:
         - FLASK_ENV=production
         - DATABASE_URL=mysql+pymysql://user:pass@mysql/ip_expert
       depends_on:
         - mysql
         - redis

     worker:
       build: .
       command: python worker.py
       depends_on:
         - redis
         - mysql
   ```

**验收标准**:
- Docker镜像构建成功
- 生产环境配置完整
- 服务启动正常

---

## 🎯 任务14: (进阶)小模型微调

### 14.1 数据准备 (第1-2天)

**目标**: 准备用于微调的训练数据集。

**负责人**: 模型团队

**具体步骤**:

1. **收集问题分类数据**
   ```python
   # 构建问题分类数据集
   classification_data = [
       {"text": "OSPF邻居建立失败", "label": "OSPF"},
       {"text": "BGP路由不通", "label": "BGP"},
       {"text": "MPLS VPN配置问题", "label": "MPLS"},
       # ... 更多数据
   ]
   ```

2. **数据清洗和标注**
   - 去重和格式统一
   - 人工标注和校验
   - 数据集划分(训练/验证/测试)

**验收标准**:
- 收集到1000+标注样本
- 数据质量高，标注准确
- 数据集划分合理

### 14.2 模型微调 (第3-5天)

**目标**: 微调BERT模型用于问题意图分类。

**负责人**: 模型团队

**具体步骤**:

1. **选择基础模型**
   ```python
   from transformers import AutoTokenizer, AutoModelForSequenceClassification

   model_name = "bert-base-chinese"
   tokenizer = AutoTokenizer.from_pretrained(model_name)
   model = AutoModelForSequenceClassification.from_pretrained(
       model_name,
       num_labels=len(categories)
   )
   ```

2. **训练脚本**
   ```python
   from transformers import Trainer, TrainingArguments

   training_args = TrainingArguments(
       output_dir="./results",
       num_train_epochs=3,
       per_device_train_batch_size=16,
       per_device_eval_batch_size=64,
       warmup_steps=500,
       weight_decay=0.01,
       logging_dir="./logs",
   )

   trainer = Trainer(
       model=model,
       args=training_args,
       train_dataset=train_dataset,
       eval_dataset=eval_dataset,
   )

   trainer.train()
   ```

**验收标准**:
- 模型训练完成
- 验证集准确率>85%
- 模型可以正常推理

---

## 📊 进度跟踪与质量控制

### 每日站会
- **时间**: 每天上午9:00
- **内容**: 汇报昨日完成情况、今日计划、遇到的问题
- **参与人**: 后端+模型团队5人 + 项目负责人

### 团队协作分工
#### 后端团队 (3人) 主要负责:
- Flask项目架构和API开发
- 数据库设计和模型定义
- 异步任务系统和队列管理
- 用户认证和权限管理
- 数据看板和统计API
- 系统部署和运维

#### 模型团队 (2人) 主要负责:
- 数据收集和IDP集成
- 向量数据库构建和配置
- 混合检索算法实现
- 提示词工程和优化
- langgraph Agent流程构建
- 模型接口集成和性能优化

#### 共同协作区域:
- `app/services/` 目录下的AI服务模块
- 异步任务的AI处理逻辑
- API接口的AI功能集成
- 系统集成测试和调试

### 代码审查
- **频率**: 每个功能模块完成后
- **标准**: 代码规范、安全性、性能、测试覆盖率
- **工具**: Git Pull Request + Code Review

### 质量检查点
1. **第一周末**: 基础架构、认证系统、数据收集和IDP集成验收
2. **第二周末**: 核心API功能、向量数据库、混合检索算法验收
3. **第三周末**: Agent流程、模型集成、完整系统集成测试验收

### 交付物清单
#### 后端交付物:
- [ ] Flask后端应用代码
- [ ] 数据库设计文档和迁移脚本
- [ ] API文档和测试用例
- [ ] 异步任务处理系统
- [ ] Docker部署配置
- [ ] 系统监控和日志配置

#### AI模型交付物:
- [ ] 处理后的知识库数据集
- [ ] Weaviate向量数据库(包含索引数据)
- [ ] 混合检索算法代码
- [ ] langgraph Agent工作流
- [ ] 提示词模板库
- [ ] 模型接口封装代码
- [ ] AI服务集成文档

### 风险预案
#### 后端风险:
1. **数据库性能问题**: 准备查询优化和索引方案
2. **异步任务堆积**: 实现任务优先级和限流机制
3. **API响应超时**: 添加缓存和异步处理

#### AI模型风险:
1. **IDP服务调用失败**: 准备本地OCR备选方案
2. **向量化API限流**: 实现请求队列和重试机制
3. **模型效果不佳**: 准备多个Prompt版本进行A/B测试
4. **OLLAMA部署问题**: 准备云端重排序API备选方案

---

## 🔧 开发环境配置

### 必需软件
- Python 3.9+
- MySQL 8.0+
- Redis 7+
- Docker & Docker Compose
- OLLAMA (用于本地模型部署)

### 开发工具
- PyCharm 或 VS Code
- Postman (API测试)
- MySQL Workbench
- Redis Desktop Manager
- Jupyter Notebook (模型实验)
- Docker Desktop

### 环境变量
```bash
export FLASK_APP=app
export FLASK_ENV=development
export SECRET_KEY=dev-secret-key
export DATABASE_URL=mysql+pymysql://root:password@localhost/ip_expert
export REDIS_URL=redis://localhost:6379

# AI服务相关 - Langchain统一集成配置
export DASHSCOPE_API_KEY=your-dashscope-api-key
export OPENAI_API_KEY=your-dashscope-api-key  # 用于langchain_openai兼容接口
export OPENAI_API_BASE=https://dashscope.aliyuncs.com/compatible-mode/v1

# 阿里云文档智能服务
export ALIBABA_ACCESS_KEY_ID=your-access-key-id
export ALIBABA_ACCESS_KEY_SECRET=your-access-key-secret

# OLLAMA本地重排序模型服务
export OLLAMA_BASE_URL=http://localhost:11434
```

---

## 📚 参考资料

### 技术文档
#### 后端技术:
- [Flask官方文档](https://flask.palletsprojects.com/)
- [SQLAlchemy文档](https://docs.sqlalchemy.org/)
- [RQ任务队列文档](https://python-rq.org/)
- [JWT认证最佳实践](https://auth0.com/blog/a-look-at-the-latest-draft-for-jwt-bcp/)

#### AI技术:
- [阿里云文档智能API文档](https://help.aliyun.com/document_detail/442515.html)
- [Weaviate官方文档](https://weaviate.io/developers/weaviate)
- [LangGraph教程](https://langchain-ai.github.io/langgraph/)
- [百炼大模型API文档](https://help.aliyun.com/zh/dashscope/)
- [OLLAMA文档](https://ollama.ai/)

### 开发规范
- [PEP 8 Python代码规范](https://pep8.org/)
- [RESTful API设计指南](https://restfulapi.net/)
- [数据库设计最佳实践](https://www.vertabelo.com/blog/database-design-best-practices/)

### 学习资源
- RAG技术原理与实践
- 向量数据库使用指南
- 提示词工程最佳实践
- 多轮对话系统设计
- LangChain开发指南

记住：保持代码质量，注重安全性，加强团队协作，及时沟通问题！🚀

