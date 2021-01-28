---
title: 基于Github的Hexo部署
date: 2021-01-28 20:57:18
categories:
- [Web,blog]
tags:
- Hexo
- Blog
- webhook
---

### 0. 服务器环境

- Ubuntu 20.04 LTS

### 1. 首先安装Hexo

```sh
#安装前置
sudo apt-get install git
sudo apt-get install nodejs
sudo apt-get install npm
sudo apt-get install nginx
#安装Hexo
npm install -g hexo-cli

#cd进储存储存博客文件的位置
cd /path/to/expamle
#在当前目录下创建文件夹"test"并初始化hexo
hexo init test

cd test
#生成网页
hexo g
```

生成后目录如下：

>test
>├── _config.landscape.yml
>├── _config.yml
>├── db.json
>├── node_modules
>├── package.json
>├── package-lock.json
>├── public
>├── scaffolds
>├── source
>│	 └── _posts
>│  		  └──  hello-world.md
>└── themes

其中node_modules用于存放依赖，scaffolds存放拼接于三种页面开头的信息

source存放已经编写好的md文件，public存放生成好的网页

### 2.利用Github实现自动部署

- 首先在github上创建仓库，用于存放source文件夹，将source文件夹存入仓库后，通过Settings - Webhooks - Add webhook添加钩子

	其中，content type选择```application/json```，Secret随意，“Which events would you like to trigger this webhook?”选择“Just the `push` event.”，勾上Active

- pip安装前置```pip3 install flask```

- 编写python脚本:

  ```python
  #"hook.py"
  import hmac
  import os
  from flask import Flask, request, jsonify
  
  app = Flask(__name__)
  # github中webhooks的secret
  github_secret = 'xxxxxxxx'
  
  def encryption(data):
      key = github_secret.encode('utf-8')
      obj = hmac.new(key, msg=data, digestmod='sha1')
      return obj.hexdigest()
  
  #'/webhook'为github中设置的URL
  @app.route('/webhook', methods=['POST'])
  def post_data():
      """
      github加密是将post提交的data和WebHooks的secret通过hmac的sha1加密，放到HTTP headers的
      X-Hub-Signature参数中
      """
      post_data = request.data
      token = encryption(post_data)
      # 认证签名是否有效
      signature = request.headers.get('X-Hub-Signature', '').split('=')[-1]
      if signature != token:
          return "token认证无效", 401
      # 运行shell脚本，更新代码
      os.system('sh deploy.sh')
      return jsonify({"status": 200})
  
  if __name__ == '__main__':
      app.run(port=8989)
  ```

- 部署nginx：

  ```nginx
  #在/etc/nginx/nginx.conf文件内http{}中添加下列配置
  server {
      listen 80; 
  
      server_name  服务器IP; # 配置域名
  
      client_max_body_size 300M;
  	
      #此处URL应与hook.py中一致
      location = /webhook { 
          proxy_pass http://127.0.0.1:8989; #端口也要保持一致
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      }
      location / {
          root /path/to/exmaple/public;
          index index.html index.htm;
      }
  }
  ```

- 将source git clone 下来，本文中仓库名为blog，clone至hexo初始化文件夹内

- 编写deploy.sh脚本与hook.py置于同一目录下

	```shell
	#!/bin/bash
	
	#拉取更新
	cd /path/to/exmaple/test/blog
	echo "------Git pulling------"
	git pull
	
	#删除原文件夹并将更新复制过去
	echo "-----Update source-----"
	\rm -rf ../source
	\cp -rf ./source ../
	
	#生成blog
	cd ..
	echo "-----Generate Blog-----"
	hexo g
	
	#重载nginx
	echo "-----Reload nginx------"
	nginx -s reload
	```

- 开启nginx，运行hook

	```shell
	systemctl nginx start
	python3 hook.py
	```

### 3. 后台运行

- 使用nohup命令

	```shell
	nohup hexo server -d >> /home/blog/output.log 2>&1 &
	```

	

### n. 关于

- 本文参考了以下文章：

	> https://blog.csdn.net/sinat_37781304/article/details/82729029
	>
	> https://www.cnblogs.com/-wenli/p/13420106.html

