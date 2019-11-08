## Flask 回顾与整理



### debugger

Flask 中的调试器拥有保护功能，通过PIN，在每次启动项目时都会生成一个PIN，当项目出现错误时，在窗口通过点击，输入PIN可以进行排除和诊断。

- 代码

  ```python
  from flask import Flask
  
  app = Flask(__name__)
  
  @app.route('/')
  def hello_world():
      return 'Hello World!'
  
  if __name__ == '__main__':
      app.run(debug=True) # 开启调试模式
  ```

  注意：如果想要修改 Environment: production，可以通过修改 vim ~/.bashrc 中添加 export FLASK_ENV = 'xxx'  即可，因为在run函数中通过会去全局变量中查找FLASK_ENV

- 在启动的时候还可以添加参数，在run函数中

  ```
  1、debug是否开启调试模式，开启后修改过后自动重启项目
  2、threaded 是否开启多线程
  3、port 启动指定服务器的端口
  4、host主机，默认是127.0.0.1，指定为0.0.0.0代表本机所有ip
  ```

  

### Flask-Script 脚本启动项目

当我们想使用类似于Django的启动方式时，需要借助于Flask-Script插件来完成，首先需要安装：

```
pip install Flask-Script
```

- app.py代码

  ```python
  from flask import Flask
  from flask_script import Manager
  app = Flask(__name__)
  
  manager = Manager(app=app)
  
  @app.route('/')
  def hello_world():
      return 'Hello World!'
  
  if __name__ == '__main__':
      manager.run() # 开启调试模式
   # 项目启动命令：python app.py runserver
  ```

  相关参数:

  ```
  -p 指定端口
  -h 指定主机
  -r 自动重新加载
  -d 开启调试模式
  --threaded 开启多线程
  
  python app.py runserver -p 8000 -h 0.0.0.0 -d -r
  ```

  

### 蓝图的出现

![1573125731234](C:\Users\wanglixing\Desktop\知识点复习\Django\gif\1573125731234.png)

源于主文件中需要加载视图函数（views.py),但是views中又需要主文件的app，所以可能出现循环引用问题。

#### 懒加载视图（借鉴于Django）

为了解决这个循环导入问题，又必须加载视图函数，所以我们可以通过懒加载的形式解决（也就是说我们通过将app以参数的形式传入，以避免重复导入）

- manage.py

  ```
  from APP import create_app
  
  app = create_app()
  
  if __name__ == '__main__':
      app.run() # 开启调试模式
  ```

  APP/\__init__.py

  ```python
  from flask import Flask
  from APP.views import init_app
  
  def create_app():
      app = Flask(__name__)
      init_app(app)  # 懒加载，其实类似于django的视图函数，将request传入
      return app
  ```

  APP/views.py

  ```python
  def init_app(app):
      @app.route('/')
      def hello_world():
          return 'Hello world'
  ```



#### 蓝图的使用

![1573127490932](C:\Users\wanglixing\Desktop\知识点复习\Django\gif\1573127490932.png)

蓝图的使用代码，很简单当成一个app来使用，最后注册即可

- APP/\__init__.py 代码如下：

  ```python
  from flask import Flask
  from APP.views import blue
  
  def create_app():
      app = Flask(__name__)
      app.register_blueprint(blue)
      return app
  ```

  APP/views.py 代码如下， manage.py不需要改动

  ```python
  from flask import Blueprint
  
  blue = Blueprint('blue',__name__)
  
  @blue.route('/')
  def index():
      return '我是蓝图的主页'
  ```

- 当有多个蓝图时，我们可以将views升级成一个包，通过在\__init__.py中导入相应的内层py来解决问题

  APP/\__init__.py

  ```python
  from flask import Flask
  from APP.views import init_view  # 初始化view
  
  def create_app():
      app = Flask(__name__)
      init_view(app)
      return app
  ```

  APP/views/\__init__.py

  ```python
  from .first_blue import blue
  from .second_blue import blue2
  
  def init_view(app):
      app.register_blueprint(blue)
      app.register_blueprint(blue2)
  ```

  APP/views/first_blue.py

  ```python
  from flask import Blueprint
  
  blue = Blueprint('blue',__name__)
  
  @blue.route('/')
  def index():
      return '我是蓝图的主页'
  ```

  APP/views/second_blue.py

  ```python
  from flask import Blueprint, render_template
  
  blue2 = Blueprint('blue2',__name__)
  
  @blue2.route('/second')
  def index():
      return render_template('index.html')
  ```

  

### ORM数据库

![1573130491960](C:\Users\wanglixing\Desktop\知识点复习\Django\gif\1573130491960.png)

在web开发中大多数的关系型数据库，ORM模型用于解决我们操作数据库的解耦合

- ORM
  - SQLAlchemy  针对于Python所有模式的一种操作数据库的模板
  - Flask-SQLAlchemy    针对于Flask开发好的一种操作数据库的模块

安装Flask-SQLAlchemy

```
pip install flask-sqlalchemy
```



- 代码演示：APP/models.py

  ```python
  from flask_sqlalchemy import SQLAlchemy
  
  models = SQLAlchemy()  # 创建一个models对象
  
  # 懒加载的方式给models添加app配置，也就是说在app创建之后再会触发配置model的app
  def init_models(app):
      models.init_app(app=app)  # models需要知道绑定的app，这里提供了懒加载的初始化配置app的方式
  
  class User(models.Model):
      id = models.Column(models.Integer,primary_key=True)
      username = models.Column(models.String(16))
  
      def __repr__(self):
          return '<User %r>' % self.username
      
      # 自定义save方法，之后就可以直接调用save了
      def save(self):
          models.session.add(self)
          models.session.commit()
  ```

  APP/\__init__.py

  ```python
  from flask import Flask
  from APP.models import init_models  # 初始化model
  from APP.views import init_view  # 初始化view
  
  def create_app():
      app = Flask(__name__)
  
      # uri 格式：数据库+驱动://用户名:密码@主机:端口/库
      app.config['SQLALCHEMY_DATABASE_URI']='sqlite:///sqlite.db' # 因为sqlite不需要指定用户名撒的，最后指定的时文件路径
      app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False # sqlalchemy 的警告，要求修改追踪关闭
      init_models(app)
      init_view(app)
      return app
  ```

  APP/views/first_blue.py

  ```python
  from flask import Blueprint
  from APP.models import models
  
  blue = Blueprint('blue',__name__)
  
  @blue.route('/')
  def index():
      return '我是蓝图的主页'
  
  @blue.route('/create_db/')
  def create_db():
      models.create_all()
      return '创建库成功'
  
  @blue.route('/drop_db/')
  def drop_db():
      models.drop_all()
      return '删除库成功'
  
  
  # 新增
  @blue.route('/adduser/')
  def add_user():
      user = User()
      user.username = 'TOM2'
  
      # 方式一
      # models.session.add(user)
      # models.session.commit()
  
      # 方式二: 自定义save方法的调用
      user.save()
  
      return '添加成功'
  ```

  

### 项目整体拆分

![1573131531857](C:\Users\wanglixing\Desktop\知识点复习\Django\gif\1573131531857.png)

下面是我们对项目的整体进行了拆分，并且针对三种开发模式，在setting.py中进行设置，以及对项目的\__init__.py进行优化后的代码:

![1573134623593](C:\Users\wanglixing\Desktop\知识点复习\Django\gif\1573134623593.png)

- 代码如下：APP/\__init__.py

  ```python
  from flask import Flask
  from APP.ext import init_ext  # 加载第三方库
  from APP.settings import envs  # 配置文件加载
  from APP.views import init_view  # 初始化view
  
  
  def create_app():
      app = Flask(__name__)
  
      # uri 格式：数据库+驱动://用户名:密码@主机:端口/库
      # app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://root:123@localhost:3306/s1' # 连接mysql
      # app.config['SQLALCHEMY_DATABASE_URI']='sqlite:///sqlite.db' # 因为sqlite不需要指定用户名撒的，最后指定的时文件路径
      # app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False # sqlalchemy 的警告，要求修改追踪关闭
  
      app.config.from_object(envs.get('develop') or 'develop') # 选择开发环境
      init_ext(app)
      init_view(app)
      return app
  ```

  APP/ext.py

  ```python
  from flask_sqlalchemy import SQLAlchemy
  models = SQLAlchemy()  # 创建一个models对象
  
  # 导入的第三方库，进行整理
  def init_ext(app):
      models.init_app(app=app)  # models需要知道绑定的app，这里提供了懒加载的初始化配置app的方式
  ```

  APP/models.py

  ```python
  from APP.ext import models
  
  class User(models.Model):
      id = models.Column(models.Integer,primary_key=True)
      username = models.Column(models.String(16))
  
      def __repr__(self):
          return '<User %r>' % self.username
  
      # 自定义save方法，之后就可以直接调用save了
      def save(self):
          models.session.add(self)
          models.session.commit()
  ```

  APP/settings.py

  ```python
  class Config:
  
      DEBUG = False
      TESTING = False
      SQLALCHEMY_TRACK_MODIFICATIONS = False
  
      @staticmethod
      def get_db_uri(db_info):
          engine = db_info.get('ENGINE') or 'sqlite'
          driver = db_info.get('DRIVER') or 'sqlite'
          user = db_info.get('USER') or ''
          password = db_info.get('PASSWORD') or ''
          host = db_info.get('HOST') or 'localhost'
          port = db_info.get('PORT') or ''
          name = db_info.get('NAME') or ''
          return '{}+{}://{}:{}@{}:{}/{}'.format(engine,driver,user,password,host,port,name)
  
  
  # 开发环境
  class DevelopConfig(Config):
      DEBUG = True
  
      db_info = {
          'ENGINE':'mysql',
          'DRIVER':'pymysql',
          'USER':'root',
          'PASSWORD':'123',
          'NAME':'s1',
          'HOST':'localhost',
          'PORT':'3306'
      }
  
      SQLALCHEMY_DATABASE_URI = Config.get_db_uri(db_info)
  
  
  # 测试环境
  class TestConfig(Config):
      DEBUG = True
  
      db_info = {
          'ENGINE':'mysql',
          'DRIVER':'pymysql',
          'USER':'root',
          'PASSWORD':'123',
          'NAME':'s1',
          'HOST':'localhost',
          'PORT':'3306'
      }
  
      SQLALCHEMY_DATABASE_URI = Config.get_db_uri(db_info)
  
  # 生产环境
  class ProductConfig(Config):
      DEBUG = False
  
      db_info = {
          'ENGINE':'mysql',
          'DRIVER':'pymysql',
          'USER':'root',
          'PASSWORD':'123',
          'NAME':'s1',
          'HOST':'localhost',
          'PORT':'3306'
      }
  
      SQLALCHEMY_DATABASE_URI = Config.get_db_uri(db_info)
  
  envs = {
      'develop':DevelopConfig,
      'testing':TestConfig,
      'product':ProductConfig,
  }
  ```

  APP/views/\__init__.py

  ```python
  from .first_blue import blue
  from .second_blue import blue2
  
  def init_view(app):
      app.register_blueprint(blue)
      app.register_blueprint(blue2)
  ```

  APP/views/first_blue.py

  ```python
  from flask import Blueprint
  from APP.models import models,User
  
  blue = Blueprint('blue', __name__)
  
  
  @blue.route('/')
  def index():
      return '我是蓝图的主页'
  
  @blue.route('/create_db/')
  def create_db():
      models.create_all()
      return '创建库成功'
  
  @blue.route('/drop_db/')
  def drop_db():
      models.drop_all()
      return '删除库成功'
  
  # 新增
  @blue.route('/adduser/')
  def add_user():
      user = User()
      user.username = 'TOM2'
  
      # 方式一
      # models.session.add(user)
      # models.session.commit()
  
      # 方式二: 自定义save方法的调用
      user.save()
  
      return '添加成功'
  ```

  

### Flask的ORM迁移 -- Flask-Migrate

当涉及到模型的字段的新增和删除时，如果没有Django的迁移，只能每次删除了新增，安装

```
pip install flask-migrate
```

- 相关代码演示：manage.py  （使得flask-migrate和flask-script结合使用）

  ```python
  from flask_migrate import MigrateCommand
  from APP import create_app
  from flask_script import Manager
  
  app = create_app()
  
  manager = Manager(app=app)
  
  manager.add_command('db',MigrateCommand)  # 添加新的脚本命令，以便于迁移
  
  if __name__ == '__main__':
      manager.run() # 开启调试模式
  ```

  APP/ext.py文件， migrate在初始化models时需要绑定app

  ```python
  from flask_migrate import Migrate
  from flask_sqlalchemy import SQLAlchemy
  
  models = SQLAlchemy()  # 创建一个models对象
  migrate = Migrate()  # 实例化一个迁移对象
  
  # 导入的第三方库，进行整理
  def init_ext(app):
      models.init_app(app=app)  # models需要知道绑定的app，这里提供了懒加载的初始化配置app的方式
      migrate.init_app(app,models)  # 使用懒加载的方式初始化配置app，绑定
  ```

  注意：结合着脚本运行

  ```
  >python manage.py db init   # 生产migration文件，记录迁移和models的更改
  >python manage.py db migrate  # 执行迁移文件
  >python manage.py db  upgrade  #将迁移文件写入数据库中
  ```

  

### 会话技术

跨请求共享数据，由于http是属于短链接；http请求是无状态的；请求从request到response就结束了

- 会话技术

  ```
  1、cookie   客户端会话技，数据存储在客户端，key-value，大小限制，对中文进行了处理支持
  2、session 
  3、token
  ```

- 常见cookie的存储使用方法

  ```python
  from flask import Blueprint, request, render_template, Response
  
  blue = Blueprint('blue', __name__)
  
  @blue.route('/login/',methods=['GET','POST'])
  def login():
      if request.method == 'GET':
          return render_template('login.html')
      elif request.method == 'POST':
          username = request.form.get('username')
          response = Response('登陆成功%s'%username)
          response.set_cookie('username',username)
          return response
      
  @blue.route('/mine/')
  def mine():
      username = request.cookies.get('username')
      return '欢迎回来%s'%username
  ```

- 存储在session中

  setting.py文件

  ```python
  class Config:
      DEBUG = False
      TESTING = False
      SQLALCHEMY_TRACK_MODIFICATIONS = False
      SECRET_KEY = 'dsferwr23@##'
  ```

  views.py文件

  ```python
  from flask import Blueprint, request, render_template, Response, session
  
  blue = Blueprint('blue', __name__)
  
  @blue.route('/login/',methods=['GET','POST'])
  def login():
      if request.method == 'GET':
          return render_template('login.html')
      elif request.method == 'POST':
          username = request.form.get('username')
          response = Response('登陆成功%s'%username)
          session['username']=username
          return response
  
  @blue.route('/mine/')
  def mine():
      username = session.get('username')
      return '欢迎回来%s'%username
  ```

  

#### Flask-Session的持久化

由于默认Flask的session是存储在客户端的，那么怎么让它存储在服务端哪？安装

```python
pip install Flask-Session
pip install redis  # 如果你要使用redis作为缓存，就得使用这个模块
```

使用起来非常简单，只需要配置SESSION_TYPE,以及实例化一个Session即可

- 代码如下：settings.py

  ```python
  class Config:
      DEBUG = False
      TESTING = False
      SQLALCHEMY_TRACK_MODIFICATIONS = False
      SECRET_KEY = 'dsferwr23@##'
      SESSION_TYPE = 'redis'  # 选择session的配置，默认会连接本地6379端口
      SESSION_COOKIE_SECURE=True
  
  ```

  ext.py文件中

  ```python
  from flask_migrate import Migrate
  from flask_session import Session
  from flask_sqlalchemy import SQLAlchemy
  
  models = SQLAlchemy()  # 创建一个models对象
  migrate = Migrate()  # 实例化一个迁移对象
  
  # 导入的第三方库，进行整理
  def init_ext(app):
      models.init_app(app=app)  # models需要知道绑定的app，这里提供了懒加载的初始化配置app的方式
      migrate.init_app(app,models)  # 使用懒加载的方式初始化配置app，绑定
      Session(app)  # 配置session
  ```

  

### Flask-Mail 发送邮件

在用户注册的时候，可能会使用到邮件注册，安装

```
pip install flask-mail
```

- 代码如下settings.py

  ```python
  ...
  # 开发环境
  class DevelopConfig(Config):
      DEBUG = True
  
      db_info = {
          'ENGINE':'mysql',
          'DRIVER':'pymysql',
          'USER':'root',
          'PASSWORD':'123',
          'NAME':'s1',
          'HOST':'localhost',
          'PORT':'3306'
      }
      # 配置mail
      MAIL_SERVER = 'smtp.163.com'
      MAIL_PORT = 25
      MAIL_USERNAME = 'python_wlx@163.com'
      MAIL_PASSWORD = 'wuyuzhu1013'
      MAIL_DEFAULT_SENDER = MAIL_USERNAME
  
      SQLALCHEMY_DATABASE_URI = Config.get_db_uri(db_info)
  ```

  使用mail，views.py中

  ```python
  from flask_mail import Message
  from APP.ext import mail
  
  @blue.route('/sendmail/')
  def send_mail():
      msg = Message('Flask-Mail测试',recipients=['python_wlx@163.com',])
      msg.html = render_template('msg.html')
      mail.send(message=msg)
      return '邮件发送成功'
  ```

  

### 用户激活

常见的提供两种策略：邮箱/短信

邮箱激活：

- 异步发送邮件
- 在邮箱中包含激活地址
  - 激活地址接受一个一次性的Token
  - Token是用户注册的时候生成的，存在于cache中
  - key-value形式： key 标识Token，value  是用户的唯一标识



短信：

- 同步操作
- 调用网易云信接口



相关代码如下：

- models.py文件中，我们使用的是自带的密码检验器，对外不公开，只能内部调用

  ```python
  from werkzeug.security import check_password_hash,generate_password_hash
  from APP.ext import models
  
  class Students(models.Model):
      id = models.Column(models.Integer,primary_key=True,autoincrement=True)
      s_name = models.Column(models.String(16),unique=True)
      _s_password = models.Column(models.String(256))
      s_phone = models.Column(models.String(11),unique=True)
  
      @property
      def s_password(self):
          raise Exception('Error Action')
  
      @s_password.setter
      def s_password(self,value):
          self._s_password = generate_password_hash(value) # 设置密码
  
      def check_password(self,password): # 校验密码的函数
          return  check_password_hash(self._s_password,password)
  ```

- views.py这边，我们使用的是网易云的短信接口，并采用缓存的形式，将短信存入redis中，以唯一标识username为key，在注册时调用

  ```python
  import hashlib
  import time
  from flask import Blueprint, request, render_template, jsonify, flash, redirect, url_for
  import requests
  from APP.ext import cache,models
  from APP.models import Students
  
  blue = Blueprint('blue', __name__)
  
  @blue.route('/register/',methods=['GET','POST'])
  def register():
      if request.method == 'GET':
          return render_template('register.html')
      elif request.method == 'POST':
          username = request.form.get('username')
          password = request.form.get('password')
          phone = request.form.get('phone')
          code = request.form.get('code')
          cache_code = cache.get(username)
          if code != cache_code:
              return '验证失败'
  
          #注册到数据库中
          student = Students()
          student.s_name = username
          student.s_password = password
          student.s_phone = phone
          models.session.add(student)
          models.session.commit()
          return 'Register Success'
  
  @blue.route('/login/',methods=['GET','POST'])
  def login():
      if request.method=='GET':
          return render_template('login.html')
  
      elif request.method=='POST':
          username = request.form.get('username')
          password = request.form.get('password')
          student = Students.query.filter(Students.s_name.__eq__(username)).first()
          print(student)
          if student and student.check_password(password):
              return 'Login Success'
          flash('用户名密码错误')
          return redirect(url_for('blue.login'))
  
  def send_verify_code(phone):
      url = 'https://api.netease.im/sms/sendcode.action'
      nonce = hashlib.new('sha512',str(time.time()).encode('utf-8')).hexdigest()
      curtime = str(int(time.time()))
      sha1 = hashlib.sha1()
      secret = '560e11c9f964'
      sha1.update((secret+nonce+curtime).encode('utf-8'))
      check_sum = sha1.hexdigest()
      header = {
          'AppKey':'f422c425f6e1a8b532d824c324804ecb',
          'Nonce':nonce,
          'CurTime':curtime,
          'CheckSum':check_sum
      }
      post_data = {
          'mobile':phone,
      }
      resp = requests.post(url,data=post_data,headers=header)
      return resp
  
  
  @blue.route('/sendcode/')
  def send_code():
      phone = request.args.get('phone')
      username = request.args.get('username')
      res = send_verify_code(phone)
      result = res.json()
      # result = {'code': 414, 'msg': 'mobile cannot be blank.'}
      if result.get('code') == 200:
          obj = result.get('obj')
          cache.set(username,obj)
          data ={
              'msg':'OK',
              'status':200,
          }
          return jsonify(data)
  
      data = {
          'msg':'fail',
          'status':400
      }
      return jsonify(data)
  ```

- 第三方库这边我们使用了缓存，配置如下：ext.py

  ```python
  from flask_migrate import Migrate
  from flask_sqlalchemy import SQLAlchemy
  from flask_caching import Cache
  
  models = SQLAlchemy()  # 创建一个models对象
  migrate = Migrate()  # 实例化一个迁移对象
  cache = Cache(
      config = {
          'CACHE_TYPE':'redis',
      }
  )
  
  # 导入的第三方库，进行整理
  def init_ext(app):
      models.init_app(app=app)  # models需要知道绑定的app，这里提供了懒加载的初始化配置app的方式
      migrate.init_app(app,models)  # 使用懒加载的方式初始化配置app，绑定
      cache.init_app(app)
  
  ```



### Flask-RESTful

正如你想的，如果设计到我们只需要提供API服务的情况下，我们根本不需要考虑template和views相关的了，只需要提供接口即可

安装：

```
pip install Flask-RESTful
```

![1573186149537](C:\Users\wanglixing\Desktop\知识点复习\Django\gif\1573186149537.png)

可插拔式：

- 我们需要创建一个apis的包：

  \__init__.py

  ```python
  from flask_restful import Api
  from APP.apis.user_api import HelloResource,UserResource
  
  api = Api()
  
  def init_api(app):
      api.init_app(app)
  
  api.add_resource(HelloResource,'/hello/')
  api.add_resource(UserResource,'/user/<int:id>/')
  ```

  user_apis.py

  ```python
  from flask_restful import Resource
  
  class HelloResource(Resource):
      def get(self):
          return {'msg':'hello api'}
  
      def post(self):
          return {'msg': 'hello post api'}
  
  class UserResource(Resource):
      def get(self,id):
          return {'msg':'user %s'%id}
  ```

- 在APP/\__init__.py

  ```python
  from flask import Flask
  from APP.ext import init_ext
  from APP.settings import envs
  from APP.apis import init_api
  
  def create_app():
      app = Flask(__name__)
      app.config.from_object(envs.get('develop') or 'develop') # 选择开发环境
      init_ext(app)
      init_api(app)  #还可以再次加载中间件
      return app
  ```

  

