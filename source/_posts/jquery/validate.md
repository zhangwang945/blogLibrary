---
title: 设计一个基于jq的表单校验插件
date: 2018-07-09 16:12:48
tags: jquery
categories: jquery
---

### 目的

学习如何去设计一个基于 jq 的表单校验插件

### 功能

支持事件触发配置、支持自定义校验规则、支持自定义校验类型

### 使用方式

默认校验

```bash
html:
    <input type="text"  validate='phone' errMsg='手机号不能为空' require><br>
    <input type="text"  validate='IDcard' errMsg='身份证不能为空' require><br/>
    <input type="text"  validate='email' errMsg='邮箱不能为空' require><br>

script:
    var validate = $('#form').validate() //校验注册
    validate() //校验
```

修改校验规则与触发事件
```bash
html:
    <input type="text"  validate='phone' errMsg='手机号不能为空' require><br>

script:
    var validate = $('#form').validate({event:'change',phone:{regular:/^138\d{8}$/}}) //校验注册
    validate() //校验
```

自定义校验类型
```bash
html:
    <input type="text"  validate='height' errMsg='身高不能为空' require><br>

script:
    var validate = $('#form').validate({event:'change',height:{regular:/^1?\d\d|2[0-5]\d$/,errMsg:'身高不合法'}}) //校验注册
    validate() //校验
```

### 代码

```bash
!function ($) {
    if (typeof $ !== 'function') {
        throw '请引入正确的jq'
    }
    $.fn.extend({
        /**
         * 表单校验(默认触发事件blur，默认配置了3个校验类型，支持扩展、规则自定义、以及自定义校验类型)
         * @param {Object} option
         * 返回一个函数 执行该函数值为true校验通过 false则未通过
         */
        validate: function (option) {
            option = $.extend({
                event: 'blur',
                phone: {
                    regular: /^1\d{10}$/,
                    errMsg: '手机号码错误'
                },
                IDcard: {
                    regular: /^\d{17[\dX]}$/,
                    errMsg: '身份证号码错误'
                },
                email: {
                    regular: /@.+\..+/,
                    errMsg: '邮箱号码错误'
                }
            }, option)
            //校验不通过的处理函数
            function err(jqEle, errorJqEle, errMsg) {
                if (errorJqEle.attr('errTip') !== undefined) {
                    errorJqEle.text(errMsg)
                } else {
                    jqEle.after(`<div style=color:red errTip>${errMsg}</div>`)
                }
                flag = false
            }
            var flag = true, _thisJq = this
            if (!(_thisJq instanceof $)) _thisJq = $(_thisJq)
            var inputs = _thisJq.find('input')
            inputs.each(function (index, ele) {
                ele = $(ele)
                var require = ele.attr('require') !== undefined, validateType = ele.attr('validate')
                var obj = option[validateType] || {}
                if (typeof obj === 'object' || require) {
                    ele.on(option.event, function () {
                        var reg = obj.regular, errorJqEle = ele.next()
                        !(reg instanceof RegExp) && (reg = new RegExp(reg))
                        if (require) {
                            if (ele.val().length === 0) {
                                err(ele, errorJqEle, ele.attr('errMsg') || '不能为空')
                                return
                            }
                        }
                        if (reg && !reg.test(ele.val())) {
                            err(ele, errorJqEle, obj.errMsg)
                            return
                        }
                        if (errorJqEle.attr('errTip') !== undefined) errorJqEle.remove()
                    })
                }
            })
            return function () {
                flag = true
                //执行input绑定的事件,校验所有的input是否通过
                if (inputs.length > 0) inputs.trigger(option.event)
                return flag
            }
        }
    })
}(jQuery)
```
