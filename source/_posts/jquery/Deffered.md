---
title: Deferred源码分析
date: 2018-07-07 16:12:48
tags: jquery
categories: jquery
---

### 目的

jquery的Deffered是es6的promise前身，是良好的异步解决方案，通过其源码学习其设计思想

### 参数、返回值

参数  ： function 可选  
返回值： deffered对象

### 使用方法

```bash
var a = $.Deferred(v => console.log(1)) //=>1
a.done(v=>console.log(v)) //添加到resolvList中
a.resolve(2) //=>2
a.then(v=>console.log(v,3)).then(v=>console.log(4)) //=>2,3    4
```

### 执行流程

大致流程如下：
1. 初始化tuples以及每个状态对应的Callbacks队列、初始化promise
2. 为promise添加done fail progress方法(Callbacks的add)
3. resolveList添加：状态更改函数、rejectedList的禁用函数、notifyList的锁定函数
rejectedList添加：状态更改函数、resolveList的禁用函数、notifyList的锁定函数
4. deffer添加 resolve | reject | notify 方法 resolveWith |rejectWith | notifyWith 方法(就是Callbacks的firwith)
5. deffer扩展promise的所有方法
6. 返回deffer

难点:promise.then实现思路：
每次都会生成一个新的deffer，利用闭包，将新的deffer各状态的更新函数添加到老的deffer各状态的队列内，以此进行状态同步,返回的是promise对象(deffer.promise)

### Deffered源码分析

```bash
Deferred: function (func) {
            var tuples = [
                    // action, add listener, listener list, final state
                    ["resolve", "done", jQuery.Callbacks("once memory"), "resolved"],
                    ["reject", "fail", jQuery.Callbacks("once memory"), "rejected"],
                    ["notify", "progress", jQuery.Callbacks("memory")]
                ],
                state = "pending",
                promise = {
                    state: function () {
                        return state;
                    },
                    always: function () {
                        deferred.done(arguments).fail(arguments);
                        return this;
                    },
                    then: function ( /* fnDone, fnFail, fnProgress */ ) { //参数为 成功、失败、处理中的回调函数  返回的是promise
                        var fns = arguments;
                        return jQuery.Deferred(function (newDefer) { //回调函数的参数是jQuery.Deferred新生成的Deffered对象
                            jQuery.each(tuples, function (i, tuple) {
                                var action = tuple[0],
                                    fn = jQuery.isFunction(fns[i]) && fns[i];
                                // deferred[ done | fail | progress ](...) for forwarding actions to newDefer  新deffer对象的状态改变的相关操作作为回调函数添加到老deffer对象中，以此进行状态同步
                                deferred[tuple[1]](function () { //defferred为老的Deffered对象 当老deffer对象已经resolve或者reject将直接执行此回调函数
                                    var returned = fn && fn.apply(this, arguments); //then对应的回调函数的执行返回值,判断其是否是deffer对象或者promise对象
                                    if (returned && jQuery.isFunction(returned.promise)) {
                                        returned.promise()
                                            .done(newDefer.resolve) //resolveList.add
                                            .fail(newDefer.reject) //rejectList.add
                                            .progress(newDefer.notify); //notifyList.add
                                    } else {
                                        //newDefer[resolveWith|rejectWith|notifyWith](...)  新deffer与老deffer状态同步
                                        newDefer[action + "With"](this === promise ? newDefer.promise() : this, fn ? [returned] : arguments);
                                    }
                                });
                            });
                            fns = null;
                        }).promise();
                    },
                    // Get a promise for this deferred
                    // If obj is provided, the promise aspect is added to the object 传参obj将会对其进行扩展promise的方法,如果不传参数将返回promise
                    promise: function (obj) {
                        return obj != null ? jQuery.extend(obj, promise) : promise;
                    }
                },
                deferred = {};

            // Keep pipe for back-compat
            promise.pipe = promise.then;

            // Add list-specific methods  变量tuples添加制定方法
            jQuery.each(tuples, function (i, tuple) {
                var list = tuple[2],
                    //当前作用域的stateString值为: resolved | rejected | undefined
                    stateString = tuple[3];

                // promise[ done | fail | progress ] = list.add  为promise添加done fail progress方法
                promise[tuple[1]] = list.add;

                // Handle state
                if (stateString) {
                    //resolve的list添加state状态变更函数、添加rejectList的disable函数、notify的list的lock
                    //reject的list添加state状态变更函数、添加resolveList的disable函数、notify的list的lock
                    list.add(function () { //为resolveList和rejectedList添加状态更改、非自己的状态的锁定、notifyList的锁定
                        // state = [ resolved | rejected ]
                        state = stateString;

                        // [ reject_list | resolve_list ].disable; progress_list.lock  i^1这里主要用于0、1交换位置(0^1=1 1^1=0)
                    }, tuples[i ^ 1][2].disable, tuples[2][2].lock);
                }

                // deferred[ resolve | reject | notify ] 为defferred添加了resolve reject notify方法
                deferred[tuple[0]] = function () {
                    //调用 resolveWith |rejectWith| notifyWith
                    deferred[tuple[0] + "With"](this === deferred ? promise : this, arguments);
                    return this;
                };
                //为defferred[resolveWith |rejectWith| notifyWith] = list.fireWith(函数队列的fireWith)  参数为上下文，伪数组参数
                deferred[tuple[0] + "With"] = list.fireWith;
            });

            // Make the deferred a promise deffered浅拷贝promise的属性方法
            promise.promise(deferred);
            // Call given func if any 如果传了回调函数 将把differ对象作为上下文和参数进行传递
            if (func) {
                func.call(deferred, deferred);
            }

            // All done!
            return deferred;
        }

```