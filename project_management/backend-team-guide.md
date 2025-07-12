# 后端开发团队任务指南

> **团队规模**: 3人  
> **核心职责**: 搭建服务架构、实现混合检索算法、知识库管理、API开发  
> **技术栈**: Python + Flask、MySQL、SQLAlchemy、RQ、Docker、JWT认证

## 📋 任务概览

### 主要任务清单
- [ ] **任务1**: Flask项目搭建与认证系统 (预计2-3天)
- [ ] **任务2**: 数据库设计与模型定义 (预计2-3天)  
- [ ] **任务3**: 核心业务API实现 (预计5-6天)
- [ ] **任务4**: 知识文档管理API (预计3-4天)
- [ ] **任务5**: 异步任务处理系统 (预计3-4天)
- [ ] **任务6**: 数据看板API实现 (预计2-3天)
- [ ] **任务7**: 系统集成与测试 (预计3-4天)

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

   # AI服务相关依赖
   langchain>=0.2.0
   langchain_openai>=0.1.0
   dashscope>=1.14.1

   # 阿里云文档智能服务
   alibabacloud_tea_openapi==0.3.7
   alibabacloud_docmind_api20220711==2.0.1

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

   # AI服务相关
   DASHSCOPE_API_KEY=your-dashscope-api-key
   OPENAI_API_KEY=your-dashscope-api-key  # 用于langchain_openai
   OPENAI_API_BASE=https://dashscope.aliyuncs.com/compatible-mode/v1

   # 阿里云文档智能服务
   ALIBABA_ACCESS_KEY_ID=your-access-key-id
   ALIBABA_ACCESS_KEY_SECRET=your-access-key-secret
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
   from alibabacloud_docmind_api20220711.client import Client
   from alibabacloud_tea_openapi import models as open_api_models
   import os
   import base64

   class IDPService:
       def __init__(self):
           config = open_api_models.Config(
               access_key_id=os.environ.get('ALIBABA_ACCESS_KEY_ID'),
               access_key_secret=os.environ.get('ALIBABA_ACCESS_KEY_SECRET'),
               endpoint='docmind-api.cn-hangzhou.aliyuncs.com'
           )
           self.client = Client(config)

       def parse_document(self, file_path):
           """
           调用阿里云文档智能API解析文档
           """
           try:
               # 读取文件并转换为base64
               with open(file_path, 'rb') as f:
                   file_content = f.read()
                   file_base64 = base64.b64encode(file_content).decode('utf-8')

               # 构建请求参数
               from alibabacloud_docmind_api20220711 import models as docmind_models

               request = docmind_models.SubmitDocumentExtractJobRequest(
                   file_name=os.path.basename(file_path),
                   file_content=file_base64
               )

               # 提交解析任务
               response = self.client.submit_document_extract_job(request)
               job_id = response.body.data.id

               # 轮询获取结果
               import time
               while True:
                   query_request = docmind_models.GetDocumentExtractJobRequest(id=job_id)
                   query_response = self.client.get_document_extract_job(query_request)

                   status = query_response.body.data.status
                   if status == 'SUCCESS':
                       return query_response.body.data.result
                   elif status == 'FAILED':
                       raise Exception(f"文档解析失败: {query_response.body.data.error_message}")

                   time.sleep(2)  # 等待2秒后重试

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

## 🎯 任务7: 系统集成与测试

### 7.1 API集成测试 (第1-2天)

**目标**: 进行完整的API集成测试。

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

### 7.2 错误处理和日志 (第2天)

**目标**: 完善错误处理和日志系统。

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

### 7.3 部署准备 (第3天)

**目标**: 准备生产环境部署配置。

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

## 📊 进度跟踪与质量控制

### 每日站会
- **时间**: 每天上午9:00
- **内容**: 汇报昨日完成情况、今日计划、遇到的问题
- **参与人**: 后端团队3人 + 项目负责人

### 代码审查
- **频率**: 每个功能模块完成后
- **标准**: 代码规范、安全性、性能、测试覆盖率
- **工具**: Git Pull Request + Code Review

### 质量检查点
1. **第一周末**: 基础架构和认证系统验收
2. **第二周末**: 核心API功能验收
3. **第三周末**: 完整系统集成测试

### 交付物清单
- [ ] Flask后端应用代码
- [ ] 数据库设计文档和迁移脚本
- [ ] API文档和测试用例
- [ ] 异步任务处理系统
- [ ] Docker部署配置
- [ ] 系统监控和日志配置

### 风险预案
1. **数据库性能问题**: 准备查询优化和索引方案
2. **异步任务堆积**: 实现任务优先级和限流机制
3. **API响应超时**: 添加缓存和异步处理

---

## 🔧 开发环境配置

### 必需软件
- Python 3.9+
- MySQL 8.0+
- Redis 7+
- Docker & Docker Compose

### 开发工具
- PyCharm 或 VS Code
- Postman (API测试)
- MySQL Workbench
- Redis Desktop Manager

### 环境变量
```bash
export FLASK_APP=app
export FLASK_ENV=development
export SECRET_KEY=dev-secret-key
export DATABASE_URL=mysql+pymysql://root:password@localhost/ip_expert
export REDIS_URL=redis://localhost:6379

# AI服务相关
export DASHSCOPE_API_KEY=your-dashscope-api-key
export OPENAI_API_KEY=your-dashscope-api-key  # 用于langchain_openai
export OPENAI_API_BASE=https://dashscope.aliyuncs.com/compatible-mode/v1

# 阿里云文档智能服务
export ALIBABA_ACCESS_KEY_ID=your-access-key-id
export ALIBABA_ACCESS_KEY_SECRET=your-access-key-secret
```

---

## 📚 参考资料

### 技术文档
- [Flask官方文档](https://flask.palletsprojects.com/)
- [SQLAlchemy文档](https://docs.sqlalchemy.org/)
- [RQ任务队列文档](https://python-rq.org/)
- [JWT认证最佳实践](https://auth0.com/blog/a-look-at-the-latest-draft-for-jwt-bcp/)

### 开发规范
- [PEP 8 Python代码规范](https://pep8.org/)
- [RESTful API设计指南](https://restfulapi.net/)
- [数据库设计最佳实践](https://www.vertabelo.com/blog/database-design-best-practices/)

记住：保持代码质量，注重安全性，及时沟通问题！🚀

