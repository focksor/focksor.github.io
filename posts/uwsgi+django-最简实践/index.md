# uWSGI+Django 最简实践


## 概况

本文档描述了使用 django 创建 web 项目，并通过 uWSGI 启动的方法和过程。


## 启动前准备
### 准备软件

本文档中涉及的软件有 `uwsgi` 和 `django`，均可以通过 pip 安装：
```shell
pip install uwsgi
pip install django
```


### 准备网站

下面我们通过 `django-admin` 创建一个最简单的应项目：
```shell
django-admin startproject mysite
```
创建好的 django 项目在 `mysite` 路径下。


## 使用 uWSGI 启动应用

使用 uWSGI 启动创建好的 django 项目：
```shell
uwsgi --http :8000 --module mysite.wsgi --chdir mysite --pidfile uwsgi.pid --daemonize uwsgi.log --vacuum
```

参数说明：

| 参数                  | 说明                                   |
| --------------------- | -------------------------------------- |
| --http :8000          | 在 8000 端口启动 HTTP 服务器           |
| --module mysite.wsgi  | 启动的应用                             |
| --chdir mysite        | 应用的路径                             |
| --pidfile uwsgi.pid   | 启动后留下 pid 文件                    |
| --daemonize uwsgi.log | 启动后在后台运行，日志输出到 uwsgi.log |
| --vacuum              | 进程结束时自动删除 pid 文件等临时文件  |

### 验证

启动后，可以使用浏览器访问 http://localhost:8000 ，可以看到 django 的欢迎页面。

除此之外，也可以使用 curl 命令访问网站：
```shell
focksor@focksor:~/workSpace/uwsgi_django_demo$ curl -s http://localhost:8000/ | grep "The install worked successfully! Congratulations!"
    <title>The install worked successfully! Congratulations!</title>
      <h1>The install worked successfully! Congratulations!</h1>
focksor@focksor:~/workSpace/uwsgi_django_demo$ 
```

### 终止
在上面我们使用后台运行的方式运行 uwsgi，并留下了 pid 文件。故我们只需要 cat 这个 pid 文件并 kill 掉对应进程就好：

```shell
kill `cat uwsgi.pid`
```

## 资源

1. [编写你的第一个 Django 应用，第 1 部分 | Django documentation | Django](https://docs.djangoproject.com/zh-hans/5.2/intro/tutorial01/)
2. [如何用 uWSGI 托管 Django | Django documentation | Django](https://docs.djangoproject.com/zh-hans/5.2/howto/deployment/wsgi/uwsgi/)
3. 本文相关脚本已上传到 Github：[focksor/uwsgi_django_demo](https://github.com/focksor/uwsgi_django_demo)

