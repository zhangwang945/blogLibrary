---
title: template
date: 2018-06-01 19:14:40
tags: javascript
categories: javascript
---

### 目的

```
学习编写简易的模板引擎
```
### 功能

```
方便快速将数据渲染到模板中,支持配置模板规则配置
```

### 使用方式

默认

```
var temp=template(`    
        hello <%= name1 %>  
    <%  list.forEach(function(val){%>
        <div><%= val %></div>
    <% })  %>
    <%- script %>`)
temp({ name1: '234', list: [12, 34], script: '<script>' })

=>    
        hello 234  
    
        <div>12</div>
    
        <div>34</div>
        
    &lt;script&gt;
```

自定义模板匹配规则

```
var temp=template(`    
        hello {{ name1 }}  
    <%  list.forEach(function(val){%>
        <div>{{ val }}</div>
    <% })  %>
    <%- script %>`,{ assign: /\{\{([\s\S]+?)\}\}/g })
temp({ name1: '234', list: [12, 34], script: '<script>' })

=>    
        hello 234  
    
        <div>12</div>
    
        <div>34</div>
        
    &lt;script&gt;
```

### 代码
```bash
<body>
    box1:
    <div id="box"></div>
    box2:
    <div id="box2"></div>
</body>
<script type="text/template" id="a">
    hello <%= name1 %>  
    <%  list.forEach(function(val){%>
        <div><%= val %></div>
    <% })  %>
    <%- script %>
</script>
<script type="text/template" id="b">
    hello {{name1}}   
    <%  list.forEach(function(val){%>
        <div>{{val}}</div>
    <% })  %>
    <%- script %>
</script>
<script>
//需要转译的html字符映射
    var escapeMap = {
        '&': '&amp;',
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&#x27;',
        '`': '&#x60;'
    };
    var esReg = new RegExp(Object.keys(escapeMap).join('|'), 'g')

    function _es(str) {
        return str === null ? '' : str.replace(esReg, function (match) {
            return escapeMap[match]
        })
    }
    function template(tmp, setting) {
        if (!setting) setting = {}
        //模板匹配规则
        var RegMap = {
            assign: setting.assign || /<%=([\s\S]+?)%>/g,
            escape: setting.escape || /<%-([\s\S]+?)%>/g,
            expression: setting.expression || /<%([\s\S]+?)%>/g
        }

        var reg = new RegExp([RegMap.assign.source, RegMap.escape.source, RegMap.expression.source].join('|'), 'g')
        var index = 0
        var funcStr = "var str=''\nwith(data){\n"
        tmp.replace(reg, function (match, assign, escape, expression, offset) {
            //截取匹配位置之前的字符
            funcStr += 'str+="' + tmp.slice(index, offset).replace(/\n/g, '\\n') + '"\n'
            //重新赋值截取字符串的起始位
            index = offset + match.length
            //处理赋值
            if (assign) {
                funcStr += 'str+=' + assign + '\n'
            } else if (escape) {//处理逃逸
                funcStr += 'str+=_es(' + escape + ')\n'
            } else if (expression) {//处理表达式
                funcStr += expression + '\n'
            }
        })
        //截取剩余字符串
        funcStr += 'str+="' + tmp.slice(index).replace(/\n/g, '\\n') + '"\n}\n return str'
        return new Function('data', funcStr)
    }
    var temp = template(a.innerHTML)
    box.innerHTML = temp({ name1: '234', list: [12, 34, 56], script: '<script>' })
        
    var temp2 = template(b.innerHTML, { assign: /\{\{([\s\S]+?)\}\}/g })
    box2.innerHTML = temp2({ name1: '234', list: [12, 34, 56], script: '<script>' })

</script>
```