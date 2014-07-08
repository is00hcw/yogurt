yogurt [ˈjoɡət]
======================

基于 [fis](http://fis.baidu.com) 扩展针对服务端为 [express.js](http://expressjs.com/) 的前端集成解决方案。

在阅读此文档之前，希望你最好对 [fis](http://fis.baidu.com) 有一定的了解。此工具主要负责前端编译与环境模拟，让你更专注、更快速地开发前端部分，关于后端 express 框架部分，请查看 [yog](https://github.com/fex-team/yog)。

## 特点

* 扩展 [swig](http://paularmstrong.github.io/swig/) 模板引擎，提供易用的 `html`、`head`、`body`、`widget`、`script`、`style` 等等标签。基于这些标签后端可以自动完成对页面的性能优化。
* 基于 `widget` 标签，可以轻松实现让 `widget` 以 BigPipe 模式渲染，详情请查看[这里](https://github.com/fex-team/yog-bigpipe)。
* 提供便利的环境、数据和页面模拟。tpl 自动与 json 数据文件关联，本地就能预览线上效果。
* 此工具负责生成 tpl 和关联的静态资源。后端只需关注页面逻辑，数据获取以及渲染模板即可，无需关心前端领域。


## 目录规范

```
├── page                    # 页面 tpl
├── static                  # 静态资源
├── widget                  # 各类组件
├── test                    # 数据与页面模拟目录
├── fis-conf.js             # fis 编译配置
├── server.conf             # 测试页面转法规则配置文件
```

### page 目录

所有页面级别的模板文件，放在此目录。此tpl 可以直接在浏览器中预览。比如 page/index.tpl 可以通过 http://127.0.0.1:8080/{{ projectname }}/page/index 访问。 其中 {{ projectname }} 为此项目的名称。

需要强调的的是，模板引擎采用的是 [swig](http://paularmstrong.github.io/swig/), 可以采用模板继承机制来实现模板复用。

layout.tpl

```tpl
<!doctype html>
{% html lang="en" framework="example:static/js/mod.js" %}
    {% head %}
        <meta charset="utf-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <meta name="description" content="">
        <meta name="author" content="">
        <link rel="icon" href="/favicon.ico">
        <title>{{ title }}</title>

        {% require "example:static/css/bootstrap.css" %}
        {% require "example:static/css/bootstrap-theme.css" %}
        {% require "example:static/css/style.css" %}
        {% require "example:static/js/bigpipe.js" %}
        {% require "example:static/js/jquery-1.10.2.js" %}
        {% require "example:static/js/bootstrap.js" %}
    {% endhead %}

    {% body %}
        <div id="wrapper">
            {% widget "example:widget/header/header.tpl" %}

            {% block beforecontent %}
            {% endblock %}
            
            <div class="container">
                {% block content %}
                {% endblock %}
            </div>

            {% block aftercontent %}
            {% endblock %}

            {% widget "example:widget/footer/footer.tpl" %}
        </div>

    {% endbody %}

    {% require "example:page/layout.tpl" %}
{% endhtml %}
```

index.tpl

```tpl
{% extends 'example:page/layout.tpl' %}

{% block beforecontent %}
    {% widget "example:widget/pagelets/jumbotron/jumbotron.tpl" %}
{% endblock %}

{% block content %}
    This is content!
{% endblock %}
```

### static 目录

用来存放所有静态资源文件，css, js, images 等等。如：

```
├── css
│   ├── bootstrap-theme.css
│   ├── bootstrap.css
│   └── style.css
├── fonts
│   ├── glyphicons-halflings-regular.eot
│   ├── glyphicons-halflings-regular.svg
│   ├── glyphicons-halflings-regular.ttf
│   └── glyphicons-halflings-regular.woff
└── js
    ├── bigpipe.js
    ├── bootstrap.js
    ├── jquery-1.10.2.js
    └── mod.js
```

### widget 目录

用来存放各类组件代码。组件分成3类。

1. 模板类：包含 tpl, 可以选择性的添加 js 和 css 文件，同名的 js 和 css 会被自动加载。

  模板类文件，可以在模板中通过 widget 标签引用。如

  ```tpl
  {% widget "example:widget/pagelets/jumbotron/jumbotron.tpl" %}
  ```
2. js 类： 主要包含 js 文件，放在此目录下的文件一般都会自动被 amd define 包裹，可选择性的添加同名 css 文件，会自动被引用。

  此类组件，可以在 tpl 或者 js 中通过 require 标签引用。

  ```tpl
  {% require('example:widget/lib/uploader/uploader.js') %}

  <script>
  var uploader = require('example:widget/lib/uploader/uploader.js');

  uploader.init();
  </script>
  ```
3. 纯 css 类：只是包含 css 文件。比如 compass. 同样也是可以通过 require 标签引用。

### test 目录。

用来存放测试数据和测试脚本。

#### 测试数据。

在你本地预览 page 目录下的 tpl 的时候，会自动在此目录下找同名的 json 文件，并将数据与之关联。

如 http://127.0.0.1:8080/example/page/index 预览的是 example 模块下，page/index.tpl 文件。如果 test/page/index.json 文件存在，则此 json 的里面的所有数据都可以在 tpl 里面使用到。

另外：此目录下还可存放其他文件，搭配 [server.conf](#server.conf) 配置，目录各类线上页面。

比如 test/data.json 文件，想通过 http://127.0.0.1:8080/testjson 页面可以访问到。只需要在 [server.conf](#server.conf) 里面配置如下内容。

```
rewrite \/testjson$ /test/example/data.json
```

#### 测试脚本

在 test 目录下的 js，也可以像 jsp, php 一样，写些动态的页面。如：

/test/ajaxResponse.js

```javascript
module.exports = function(req, res, next) {
    res.write('Hello world');

    res.setHeader('xxxx', 'xxx');

    res.end('The time is ' + Date.now());
};
```

配合 [server.conf](#server.conf) 可以用来模拟线上的 ajax 响应页面。

```
rewrite \/ajax$ /test/example/ajax.js
```

这样当你请求 http://127.0.0.1:8080/ajax 页面的时候，就会自动执行这个脚本 js.

### fis-conf.js 

编译配置文件，详情请查看[配置 API](http://fis.baidu.com/docs/api/fis-conf.html)。

### server.conf

支持 rewrite 和 redirect 两条指令。语法非常简单。

```
指令名称  正则  目标地址
```

如：

```
# 首页
rewrite \/$ /example/page/index

# json 文件
rewrite \/json /example/data.json

# 自定义页面。
rewrite \/ajax /example/ajax.js
```

非 rewrite, redirect 打头的行，都被认为是注释。
