###  此项目是练习将django和Vue结合到一起的一种方式。

[原文链接](https://zhuanlan.zhihu.com/p/25080236?hmsr=pycourses.com&utm_source=pycourses.com&utm_medium=pycourses.com)



django-admin startproject ulb_manager

tree ulb_manager

cd ulb_manager/ 

python manage.py startapp backend

### #使用vue1.0版本
vue init webpack#1.0 frontend   都选择默认即可

cd frontend/

npm install

npm run build


使用Django的通用视图 TemplateView
找到项目根 urls.py (即ulb_manager/urls.py)，使用通用视图创建最简单的模板控制器，访问 『/』时直接返回 index.html


	from django.views.generic import TemplateView
	
	urlpatterns = [
	
	    url(r'^admin/', admin.site.urls),
	    url(r'^$', TemplateView.as_view(template_name="index.html")),
	    url(r'^api/', include('backend.urls', namespace='api'))
	]



配置Django项目的模板搜索路径
上一步使用了Django的模板系统，所以需要配置一下模板使Django知道从哪里找到index.html

打开 settings.py (ulb_manager/settings.py)，找到TEMPLATES配置项，修改如下:


	TEMPLATES = [
	    {
	        'BACKEND': 'django.template.backends.django.DjangoTemplates',
	        # 'DIRS': [],
	        **'DIRS': ['frontend/dist']**,
	        'APP_DIRS': True,
	        'OPTIONS': {
	            'context_processors': [
	                'django.template.context_processors.debug',
	                'django.template.context_processors.request',
	                'django.contrib.auth.context_processors.auth',
	                'django.contrib.messages.context_processors.messages',
	            ],
	        },
	    },
	]
注意这里的 frontend 是VueJS项目目录，dist则是运行 npm run build 构建出的index.html与静态文件夹 static  的父级目录

这时启动Django项目，访问 / 则可以访问index.html，但是还有问题，静态文件都是404错误，下一步我们解决这个问题

###  配置静态文件搜索路径
打开 settings.py (ulb_manager/settings.py)，找到 STATICFILES_DIRS 配置项，配置如下:

		# Add for vuejs
		STATICFILES_DIRS = [
		    os.path.join(BASE_DIR, "frontend/dist/static"),
		]
这样Django不仅可以将/ulb 映射到index.html，而且还可以顺利找到静态文件

此时访问 /ulb 我们可以看到使用Django作为后端的VueJS helloworld






cd ulb_manager/ 

python manage.py runserver 0.0.0.0:18888

打开浏览器，输入地址127.0.0.0:18888即可看到页面。


# 开发环境  

因为我们使用了Django作为后端，每次修改了前端之后都要重新构建（你可以理解为不编译不能运行）

npm run dev

但是有个新问题，使用VueJS的开发环境脱离了Django环境，访问Django写的API，出现了跨域问题，有两种方法解决，一种是在VueJS层上做转发（proxyTable)(关于转发的设置问题可以在另一个项目中查看)。另一种是在Django层注入header，这里我使用后者，用Django的第三方包 django-cors-headers 来解决跨域问题


安装
pip install django-cors-headers


配置（两步）

1. settings.py 修改


		MIDDLEWARE = [
		    'django.middleware.security.SecurityMiddleware',
		    'django.contrib.sessions.middleware.SessionMiddleware',
		    **'corsheaders.middleware.CorsMiddleware',**
		    'django.middleware.common.CommonMiddleware',
		    'django.middleware.csrf.CsrfViewMiddleware',
		    'django.contrib.auth.middleware.AuthenticationMiddleware',
		    'django.contrib.messages.middleware.MessageMiddleware',
		    'django.middleware.clickjacking.XFrameOptionsMiddleware',
		]
这里要注意中间件加载顺序，列表是有序的哦



2. settings.py 添加

		CORS_ORIGIN_ALLOW_ALL = True



至此，我的开发环境就搭建完成了



#### 部署：


pip install uwsgi
apt-get install nginx


uwsgi配置文件：

	
	[uwsgi]
	socket = 127.0.0.1:9292
	stats = 127.0.0.1:9293
	workers = 4
	# 项目根目录
	chdir = /opt/inner_ulb_manager
	touch-reload = /opt/inner_ulb_manager
	py-auto-reload = 1
	# 在项目跟目录和项目同名的文件夹里面的一个文件
	module= inner_ulb_manager.wsgi
	pidfile = /var/run/inner_ulb_manager.pid
	daemonize = /var/log/inner_ulb_manager.log



nginx 配置文件：


	server {
	    listen 8888;
	    server_name 120.132.**.75;
	    root /opt/inner_ulb_manager;
	    access_log /var/log/nginx/access_narwhals.log;
	    error_log /var/log/nginx/error_narwhals.log;
	
	    location / {
	            uwsgi_pass 127.0.0.1:9292;
	            include /etc/nginx/uwsgi_params;
	    }
	    location /static/ {
	            root  /opt/inner_ulb_manager/;
	            access_log off;
	    }
	    location ^~ /admin/ {
	            uwsgi_pass 127.0.0.1:9292;
	            include /etc/nginx/uwsgi_params;
	    }
	}



/opt/inner_ulb_manager/static 即为静态文件目录，那么现在我们静态文件还在 frontend/dist 怎么办，不怕，Django给我们提供了命令：

先去settings里面配置：


	STATIC_ROOT = os.path.join(BASE_DIR, "static")
然后在存在manage.py的目录，即项目跟目录执行：


	python manage.py collectstatic
这样frontend/dist/static里面的东西就到了项目根目录的static文件夹里面了


那么我们专门为Django创建一个生产环境的配置 prod.py

prod.py 与 默认 settings.py 同目录


	# 导入公共配置
	from .settings import *
	
	# 生产环境关闭DEBUG模式
	DEBUG = False
	
	# 生产环境开启跨域
	CORS_ORIGIN_ALLOW_ALL = False
	
	# 特别说明，下面这个不需要，因为前端是VueJS构建的，它默认使用static作为静态文件入口，我们nginx配置static为入口即可，保持一致，没Django什么事
	STATIC_URL = '/static/'

如何使用这个配置呢，进入 wisg.py 即uwsgi配置里面的module配置修改为：
	import os
	
	from django.core.wsgi import get_wsgi_application
	
	os.environ.setdefault("DJANGO_SETTINGS_MODULE", "**inner_ulb_manager.prod**")
	
	application = get_wsgi_application()
启动uwsgi


	uwsgi --ini inner_ulb_manager.ini

启动ngingx


	service nginx start

至此，部署就完成了




