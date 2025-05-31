# uWSGI+CGI 最简实践



## 概况
uWSGI 提供了一个 cgi 插件，可以让你通过 uWSGI 来运行传统的 CGI 脚本（虽然现在一般不会有人这样做）。

 ⚠️ 注意事项
- uWSGI 的 CGI 插件性能一般，并不适合高并发或生产环境，只适合兼容旧系统或做一些调试。
- 它只是提供了一个兼容层，内部仍是 fork+exec 来执行 CGI 脚本，性能开销与传统 CGI 接近。
- uWSGI 更多的是用于运行 WSGI 应用，如 Flask、Django，而不是传统 CGI。

## 安装 uWSGI
### 编译安装 uWSGI+CGI plugin
uWSGI 默认不支持 CGI 应用。要支持 CGI，则需要安装其 CGI plugin。通过以下命令编译 uWSGI 及其 CGI 补丁：
```shell
git clone git@github.com:unbit/uwsgi.git -b uwsgi-2.0 --depth=1
cd uwsgi
make PROFILE=cgi
python uwsgiconfig.py --plugin plugins/cgi
```

编译完成后，通过以下命令验证 uWSGI 的可用性：
```shell
$ ls ./cgi_plugin.so
./cgi_plugin.so
$ ./uwsgi --plugin cgi --plugins-list 2>&1 | grep cgi
9: cgi
```
如上所示，uWSGI 以及其 cgi 插件已经就绪。

## 编写 CGI 应用
CGI 应用说白了就是一个可执行的程序，启动后能够输出 HTTP 响应内容就好了。下面是使用 C 和 BASH 实现的 CGI 应用。

### 使用 C 创建 CGI 应用
创建 `hello.c`：
```c
#include <stdio.h>

int main(void)
{
    printf("Content-type: text/plain\n\n");
    printf("Hello, CGI world!\n");
    return 0;
}
```

将其编译为 cgi 应用，编译后尝试执行，可以输出内容：
```shell
$ gcc hello.c -o apps/hello.cgi 
$ ./apps/hello.cgi 
Content-type: text/plain

Hello, CGI world!
```


### 使用 Bash 创建 CGI 应用
编写 `bash.cgi`
```bash
#!/bin/bash

echo "Content-Type: text/plain"
echo ""
echo "Hello, CGI Bash Script!"
```

创建后给它赋予执行权限，并尝试执行，可以输出内容：
```shell
$ chmod +x apps/bash.cgi 
$ ./apps/bash.cgi 
Content-Type: text/plain

Hello, CGI Bash Script!
```

## 启动 uWSGI
通过以下命令启动 uWSGI 应用：
```shell
./uwsgi --plugin cgi --cgi ../uwsgi_simple_app/apps/ --http-socket :9000 --http-socket-modifier1 9
```
参数说明：

| 参数                            | 说明                                                                                                                                            |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| --plugin cgi                    | 加载 cgi plugin                                                                                                                                 |
| --cgi ../uwsgi_simple_app/apps/ | 指定 cgi 应用所在的路径                                                                                                                         |
| --http-socket :9000             | 使用 HTTP 协议，绑定到 9000 端口                                                                                                                |
| --http-socket-modifier1 9       | 使用 CGI 补丁处理 HTTP 协议的请求。<br><br>其中 9 是 CGI 的标识，可以用如下命令看到：<br>`./uwsgi --plugin cgi --plugins-list 2>&1 \| grep cgi` |

### 验证访问
uWSGI 启动后，可以通过 curl 或浏览器访问相应的 url 以访问应用。下面以 curl 为例：
```shell
$ curl http://localhost:9000/hello.cgi
Hello, CGI world!
$ curl http://localhost:9000/bash.cgi
Hello, CGI Bash Script!
```

## 易踩坑问题
#### 未加载应用
如果启动日志中输出以下内容：
```log
*** no app loaded. GAME OVER ***
```
这是因为 CGI 应用是动态加载的，所以会显示没有加载应用，上面的日志是正常的。

但是如果启动日志中输出以下内容
```log
*** no app loaded. GAME OVER ***
```
则需要在启动参数中加入 `--need-app=false` (或删除 `--need-app=true`，如果你加了的话)。


### 未找到标识符 0
出现该问题时，启动日志中输出以下内容：
```log
-- unavailable modifier requested: 0 --
```
且 curl 或浏览器无法访问 uwsgi 绑定的服务器地址。这是因为 uwsgi 默认是以 Python 插件（标识符为 0）处理的，如果你的 uWSGI 并没有编译这个插件，则会报这个错误。

解决方法：

在我们的应用场景中（使用 CGI 插件处理 HTTP 请求），是完全不用 Python 插件的。因此，当出现该问题的时候，你只需要在启动命令加上参数 `--http-socket-modifier1 9`，要求 uWSGI 以 CGI 插件（标识符为 9）处理 http-socket 请求就好了。
> 如果你不是通过 `--http-socket` 参数的方式绑定监听地址，则使用 `./uwsgi --help | grep modifier1` 查看并找到符合你需求的参数。

但是，如果你确实是想要以 Python 插件启动应用时遇到了这个问题，则是因为你的 uWSGI 并没有集成 Python 插件，使用以下命令重新编译 uWSGI 以解决问题：
```shell
make clean
make PROFILE=default
```

### 未找到 Python 应用
出现该问题时，启动日志中输出以下内容：
```log
--- no python application found, check your startup logs for errors ---
```

这种情况是因为你的 uWSGI 中已经集成了 Python 插件，当 uWSGI 启动时会默认去查找是否有 Python web 应用（很明显在我们这篇文档所述的环境里是找不到的），因此会提示此错误。

这个问题出现的原因其实跟上个问题是一样的，错误信息仅仅取决于你的 uWSG 是否集成了 Python 插件。要解决这个问题，我们需要添加一个参数来切换默认的处理应用：`--http-socket-modifier1 9`（同上个问题的解决方法）。

### 找不到 CGI 插件
出现该问题时，启动日志中输出以下内容：
```log
open("./cgi_plugin.so"): No such file or directory [core/utils.c line 3709]
!!! UNABLE to load uWSGI plugin: ./cgi_plugin.so: cannot open shared object file: No such file or directory !!!
```

这是因为你没有编译 CGI 插件。使用以下命令编译插件以解决问题：
```shell
python uwsgiconfig.py --plugin plugins/cgi
```


## 资源
1. 上文中所有提及的源码及其验证脚本打包上传到了 [Github](https://github.com/focksor/simple_uwsgi_cgi_apps) 上，希望对你有帮助。
2. uWSGI CGI 官方文档：[Running CGI scripts on uWSGI — uWSGI 2.0 documentation](https://uwsgi-docs.readthedocs.io/en/latest/CGI.html)

