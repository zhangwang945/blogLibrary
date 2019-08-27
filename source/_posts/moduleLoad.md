---
title: 模块加载器
date: 2018-08-01 10:14:30
tags: javascript
categories: javascript
---

### 目的

```
学习如何实现模块加载器
```

### 原理

![示例图片](https://t1.picb.cc/uploads/2019/08/26/gbBNze.png "示例图片")

具体思路
1.定义状态对象status标识不同状态对应的状态码
2.定义module模型关键属性，uri 资源定位,deps 该模块依赖,waitings 依赖该模块的模块,remain 未加载依赖的数量
3.通过当前文档的路径document.URL解析出当前文档的路径
4.解析依赖路径，并加载，加载完的依赖模块继续判断其是否还有依赖，如果有将继续加载；如没有，则通知依赖于该模块的module，该依赖项加载完：通过waitings找到对应依赖于该模块的module使其remain--，当remain为0时，说明当前模块依赖已经加载完，执行该模块函数并导出结果，继续通知依赖于该模块的module，如此按照类递归的方式直到全部加载完


### 代码
```bash
//入口启动
//分析下载资源的依赖
(function (global) {
  var startUp = { version: "1.00" };
  global.startUp = startUp;
  var status = {
    FETCHED: 1,
    LOADED: 1
  };
  var cache = {};
  // 提取当前文件的路径
  var relativePathReg = /[^?]*\//;
  cache.preload = [];
  var docUrl = document.URL.match(relativePathReg)[0];
  var jsMeta = {};
  var cid = 0;
  function getCid() {
    return cid++;
  }
  // 解析依赖路径
  function resolve(list, uri) {
    var arr = [];
    (list || []).forEach(fileName => {
      arr.push(uri + fileName);
    });
    return arr;
  }
  // 读取缓存
  function getModule(uri, deps) {
    var m = cache[uri];
    if (!m) m = cache[uri] = new Module(uri, deps);
    return m;
  }
  // 请求js文件
  function request(url, callback) {
    var script = document.createElement("script");
    script.src = url;
    script.onload = function () {
      callback();
      document.body.removeChild(script);
    };
    document.body.appendChild(script);
  }

  //模块构造函数
  function Module(uri, deps) {
    this.uri = uri;
    this.status = 0;
    this.deps = deps || [];
    this.remain = this.deps.length; //记录依赖的未下载项
    this.waitings = {};//依赖于该模块的模块
    this.exports = null;
  }
  // 加载依赖
  Module.prototype.load = function () {
    var module = this;
    var parentUri = module.uri;
    // 遍历依赖
    module.deps.forEach(depUrl => {
      var m = getModule(depUrl);
      m.waitings[parentUri] = 1;
      // 检查是否下载过
      if (m.status < status.FETCHED) {
        request(depUrl, function () {
          m.deps = resolve(jsMeta.deps, m.uri.match(relativePathReg)[0]);
          m.factory = jsMeta.factory;
          m.remain = m.deps.length;
          m.load();
          jsMeta = {};
        });
        m.status = status.FETCHED;
      } else {
        module.remain--;
      }
    });
    if (module.remain === 0) module.onload();
  };
  // 加载完依赖后，执行本模块（说明依赖已经完成）
  Module.prototype.onload = function () {
    if (this.entryFactory) {
      var args = this.deps.map(url => getModule(url).exports);
      this.entryFactory.apply(null, args);
    }
    this.exec();
    var waitings = this.waitings;
    var pUri;
    for (pUri in waitings) {
      var pMod = getModule(pUri);
      pMod.remain--;
      // 当前模块依赖加载完则逐级向上
      if (pMod.remain === 0) pMod.onload();
    }
  };
  // 初始化require函数
  Module.prototype.genRequire = function () {
    var module = this;
    return function (uri) {
      return getModule(module.uri.match(relativePathReg)[0] + uri).exports;
    };
  };
  // 执行模块的函数，保存return值
  Module.prototype.exec = function () {
    var module = this;
    var factory = module.factory;
    if (!factory) return;
    var exports = (module.exports = {});
    var require = module.genRequire();
    exports = module.factory(require, exports, module);
    if (exports === undefined) {
      exports = module.exports;
    } else {
      module.exports = exports;
    }
    return exports;
  };
  // 提取依赖的正则
  var extractDepReg = /\/\/.*|\/\*[\s\S]*\*\/|\brequire\(\s*(["'])(.+?)\1\s*\)/g;//"
  // 模块定义
  Module.define = function (factory) {
    var deps = [];
    var factoryStr = factory.toString();
    factoryStr.replace(extractDepReg, function (match, dot, uri) {
      if (uri) deps.push(uri);
    });
    jsMeta.deps = deps;
    jsMeta.factory = factory;
  };
  Module.use = function (list, callback, uri) {
    var deps = resolve(list, docUrl);
    var m = getModule(uri, deps);
    m.entryFactory = callback;
    m.status = status.FETCHED;
    m.load();
    // console.log(m);
  };
  // 启动
  startUp.use = function (list, callback) {
    var preload = cache.preload;
    // 检查是否有预加载的配置
    if (preload.length) {
      // 预加载的文件
      Module.use(
        preload,
        function () {
          Module.use(list, callback, docUrl + getCid());
        },
        docUrl + getCid()
      );
    } else {
      Module.use(list, callback, docUrl + getCid());
    }
  };
  startUp.preload = function (list) {
    cache.preload = list;
  };
  global.define = Module.define;
})(this);


```
