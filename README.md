yogurt [ˈjəʊgət]
======================

node 服务端框架与相应前端 F.I.S 解决方案设计文档。

整体开发分两个流程，前端开发与后端开发。前端开发主要集中在 html 和 js 的编写上，所有页面和数据通过 fis 模拟协助快速开发。后端开发主要集中在模板渲染和提供数据等逻辑上。

采用前后端分离主要是考虑到更好的分工合作，同时通过页面与数据模拟可以大大的减少前端开发成本，达到快速开发的效果。

## 前端篇

### 目录规范

```bash
├── page                    # 页面 tpl
├── static                  # 静态资源
├── widget                  # 各类组件
├── test                    # 数据与页面模拟目录
├── fis-conf.js             # fis 编译配置
```

### 模板

基于 [swig](http://paularmstrong.github.io/swig/) 扩展 html、head、body、style、script、require、uri、widget 等标签，方便组织代码和静态资源引用，自动完成 js、css 优化输出。

layout.html

```tpl
<!doctype html>
{% html lang="en" framework="/static/js/mod.js" %}
    {% head %}
        <meta charset="UTF-8">
        <title>{% title | escape %}</title>
        {% require "common:static/js/jquery.js" %}
        
        {% style %}
            body { color: white;}
        {% endstyle %}
        
        {% style %}
            console.log('hello yogurt');
        {% endstyle %}
    
    {% endhead %}

    {% body %}
        <div id="wrap">
            {% block content %}
            This will be override.
            {% endblock %}
        </div>
    {% endbody %}
{% endhtml %}
```

index.html

```tpl
{% extends 'layout.html' %}

{% block content %}
<p>This is just an awesome page.</p>
{% endblock %}
```

### widget 模块化

页面中通用且独立的小部分可以通过 widget 分离出来，方便维护。

widget/header/header.html

```tpl
<div class="header">
    <ul>
        <li>nav 1</li>
        <li>nav 2</li>
    </ul>
</div>
```

page/index.html

```tpl
{% extends 'layout.html' %}

{% block content %}
    {% widget "widget/header/header.html" %}
{% endblock %}
```

### widget 渲染模式

借鉴了 BigPipe，Quickling 等思路，让 widget 可以以多种模式加载。

1. `sync` 默认就是此模式，直接输出。
2. `quicking` 此类 widget 在输出时，只会输出个壳子，内容由用户自行决定通过 js，另起请求完成填充，包括静态资源加载。
3. `async` 此类 widget 在输出时，也只会输出个壳子，但是内容在 body 输出完后，chunk 输出 js 自动填充。widget 将忽略顺序，谁先准备好，谁先输出。
4. `pipeline` 与 `async` 基本相同，只是它会按顺序输出。

```tpl
{% extends 'layout.html' %}

{% block content %}
    {% widget "widget/header/header.html" mode="pipeline" %}
{% endblock %}
```

## 后端篇

### 目录规范

```bash
├── config                    # 配置文件
│   ├── UI-A-map.json         # 静态资源表。
│   ├── UI-B-map.json         # 静态资源表。
│   ├── config.json           # 默认配置
│   └── development.json      # 开发期配置项，便于调试。
├── controllers               # 控制器
│   └── ...                   # routes
├── locales                   # 多语言
├── models                    # model
│   └── ...
├── public
│   │── UI-A                  # UI-A 的所有静态资源
│   │   └── ...
│   └── UI-B                  # UI-B 的所有静态资源
│       └── ...
├── views
│   │── UI-A                  # UI-A 的模板文件。
│   │   └── ...
│   └── UI-B                  # UI-B 的模板文件。
│       └── ...
└── server.js                 # server 入口
```

###  服务端处理流程图

![workflow](./flow.jpg)

**中间件说明**

* `compress` gzip 传输内容。
* `favicon` 其实就是一个静态文件，跟 `static` 分开是因为它可以做更长的缓存。
* `static` 设定静态目录，处理静态文件。
* `logger` 对 http 请求做日志记录。
* `bodyparser` 对不同的发送方式做解析，如：form-data, multipart, json.
* `cookie` 解析 cookie 数据，方便后续 app 使用。
* `session` 类似于 `cookie`, 数据存储于服务端。
* `security` 做服务端安全处理，防止入侵。
* `Fis Source Map`  解析前端模块生成的静态资源表，帮助后续 app 定位静态资源。
* `i18n` 多语言数据读取，帮助后续 app 或模板对多语言的支持。
* `BigPipe` 帮助后续 app 和 view 层实现快速渲染功能。
* `Router` 分组多种路由，集成在独立的特定文件上维护。

### Fis 静态资源定位

集成在模板引擎中，通过使用扩展的 custom tags 便能正确的资源定位。此功能主要依赖与前端模块生成的静态资源表。借助静态资源表，我们可以简单的实现将静态资源部署在 cdn 或者其他服务器上。

### router & controllers

主要起到一个组的概念，将多个路由，按照相同的前缀分类。

比如原来你可能需要这么写。

```javasript
app.get('/user', function(req, res, next) {
    res.render('user/list.tpl', data);
});

app.get('/user/add', function(req, res, next) {
    // todo
});

app.get('/user/edit/:id', function(req, res, next) {
    // todo
});

app.get('/user/read/:id', function(req, res, next) {
    // todo
});
```

此 router 中间件可以让你把 user 操作的一系列路由，组合写在一个 user.js 文件里面，效果与上面代码完全相同。

controllers/user.js

```javascript
module.exports = function(router) {
    router.get('/list', function(req, res, next) {
        // todo
    });

    router.get('/add', function(req, res, next) {
        // todo
    });

    router.get('/edit/:id', function(req, res, next) {
        // todo
    });

    router.get('/read/:id', function(req, res, next) {
        // todo
    });
};
```

### BigPipe

为了更快速的呈现页面, 可以让页面整体框架先渲染，后续再填充内容。更多请查看[widget 渲染模式](#widget 渲染模式)。

其实对于页面渲染过程中，会拖慢渲染的主要是 model 层数据获取。传统的渲染模式 `res.render(tpl, data)`, 都是先把数据都准备好了才开始渲染，这样其实并没有避开用户等待。

现在的方式是 `res.render()` 只准备框架必要的数据，等框架渲染完后，开始渲染 widget，在渲染之前通过 callback 与 controller 打交道，补充绑定 widget 的数据, 等数据 ready 再开始完成 widget 渲染。这样便能减少用户的等待时间。

```javascript
router.get('/', function(req, res) {
   
   var frameModel = {
       title: 'xxxx',
       navs: [{}, {}]
    }

    req.bigpipe
        .bind('pageletId', function( done ) {
            // 此方法会在 widget 在渲染前触发。
            // widget 可能会在 body 输出完后渲染，
            // 也可能会在下次请求的时候开始渲染。

            var user = new User();
        
            user.fetch();

            user.then(function( value ) {
                done( value );
            });
        })
        .render('user.tpl', frameModel);
});
```

### model 管理器

为了方便 widget 与 model 关联，实现简单通过配置项就能完成的能力，需要一个集中管理 models 的管理器。同时，为了更方便的控制 model 的生命周期，我们也可以通过这个管理器来维护。

```javascript
var user = ModelFactory.get('user', 'session');

user.then(function(val) {
    res.render(tpl, val);
});

user.init();
```

```tpl
...
{% widget "widget/header/header.html" mode="pipeline" model="user" scope="session" %}
...
```

如上面 widget 将通过 `pipeline` 方式渲染。 即当框架输出完成后，自动创建名字为 `user`
 的 model，由于 scope="session" 所以同一个 session 期只会创建一个实例。等 model 数据到位再把内容吐出来完成整个 widget 的渲染。其他渲染模式也类似。

 scope 说明

 1. `request` 生命周期为请求期，即没有一个请求都会创建一个 model。
 2. `session` 生命周期为 session。即来自同一个用户，短期内的所有请求只会创建一个 model
 3. `application` 生命周期为整个应用程序的生命周期，即整个 server 从开始到结束只会创建一个 model.
 