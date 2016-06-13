---
layout: post
title: "利用Flask的RESTful插件简化API开发"
date: 2014-03-12 13:45:07 +0800
comments: true
categories: Python, Flask
---
今天尝试用Flask的RESTful插件改造昨天写的API，使之更清晰明了。
<!--more-->
## 准备工作
进入昨天创建的虚拟开发环境venv，  
```
. venv/bin/activate
```
安装flask-restful，
```
pip install flask-restful
```

## 开始改造
### 代码基础机构
首先搭好Flask-Restful的代码框架，然后往里面填写代码，避免思路被打断。

在app = Flask(__name__)下方加入一行api = Api(app)，该api实例将用来注册路由。

``` python
from flask import Flask, jsonify, abort, request, url_for
from flask.ext.httpauth import HTTPBasicAuth
from flask.ext.restful import Api, Resource
 
app = Flask(__name__)
api = Api(app)
auth = HTTPBasicAuth()
 
@auth.get_password
def get_password(username):
    if username == 'linkoubian':
        return '7c4a8d09ca3762af61e59520943dc26494f8941b'
    return None
 
@auth.error_handler
def unauthorized():
    return make_response(jsonify( { 'error': 'Unauthorized access' } ), 403)
    # return 403 instead of 401 to prevent browsers from displaying the default auth dialog

tasks = [
    {
        'id': 1,
        'title': u'Buy groceries',
        'description': u'Milk, Cheese, Pizza, Fruit, Tylenol', 
        'done': False
    },
    {
        'id': 2,
        'title': u'Learn Python',
        'description': u'Need to find a good Python tutorial on the web', 
        'done': False
    }
]

class TaskListAPI(Resource):
    def get(self):
        pass

    def post(self):
        pass

class TaskAPI(Resource):
    def get(self, id):
        pass

    def put(self, id):
        pass

    def delete(self, id):
        pass

api.add_resource(TaskListAPI, '/todo/tasks', endpoint = 'tasks')
api.add_resource(TaskAPI, '/todo/tasks/<int:id>', endpoint = 'task')
   
if __name__ == '__main__':
    app.run(debug = True)
```

### 提取验证逻辑
然后为TaskListAPI及TaskAPI这两个资源分别添加初始化方法，以处理数据验证等。

``` python TaskListAPI的初始化方法
    def __init__(self):
        self.reqparse = reqparse.RequestParser()
        self.reqparse.add_argument('title', type = str, required = True, help = 'No task title provided', location = 'json')
        self.reqparse.add_argument('description', type = str, default = "", location = 'json')
        super(TaskListAPI, self).__init__()
```

``` python TaskAPI的初始化方法
    def __init__(self):
        self.reqparse = reqparse.RequestParser()
        self.reqparse.add_argument('title', type = str, location = 'json')
        self.reqparse.add_argument('description', type = str, location = 'json')
        self.reqparse.add_argument('done', type = bool, location = 'json')
        super(TaskAPI, self).__init__()
```

注意RequestParser默认是从request.values中取值的，所以此处我们要指定location为json，以从request.json中解析传递过来的数据。

### 重构新增task的代码
``` python 旧的代码
def create_task():
    if not request.json or not 'title' in request.json:
        abort(400)
    task = {
        'id': tasks[-1]['id'] + 1,
        'title': request.json['title'],
        'description': request.json.get('description', ""),
        'done': False
    }
    tasks.append(task)
    return jsonify( { 'task': make_public_task(task) } )
```

``` python 新的代码
    def post(self):
        args = self.reqparse.parse_args()
        task = {
            'id': tasks[-1]['id'] + 1,
            'title': args['title'],
            'description': args['description'],
            'done': False
        }
        tasks.append(task)
        return { 'task': make_public_task(task) }
```

不光数据验证相关的代码被移除了，description的默认值也不必在新代码里体现。由于Flask-Restful会自动将数据转成JSON格式，所以jsonify函数也可不必调用。

另外，Flask-Restful插件提供的marshal功能，可以优雅的完成make_public_task的功能。只需定义一个数据模板，
``` python
task_fields = {
    'id': fields.Integer,
    'title': fields.String,
    'description': fields.String,
    'done': fields.Boolean,
    'uri': fields.Url('task', absolute=True)
}
```
然后将make_public_task(task)替换成marshal(task, task_fields)即可。

### 重构更新task的代码
原来代码中有大段的验证逻辑，现在可以统统去掉，只需如下编写方法即可，
``` python
    def put(self, id):
        task = filter(lambda t: t['id'] == id, tasks)
        if len(task) == 0:
            abort(404)
        task = task[0]
        args = self.reqparse.parse_args()
        for k, v in args.iteritems():
            if v != None:
                task[k] = v
        return { 'task': marshal(task, task_fields) }
```

## 说明
本文是根据[Miguel Grinberg](http://blog.miguelgrinberg.com/post/about-me)写的这篇[文章](http://blog.miguelgrinberg.com/post/designing-a-restful-api-using-flask-restful)翻译并整理的。