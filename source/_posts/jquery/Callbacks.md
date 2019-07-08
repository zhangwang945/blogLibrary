---
title: Callbacks源码分析
date: 2019-07-07 16:12:48
tags: jquery
categories: jquery
---

### 目的

```
jquery的Callbacks函数队列设计是很出彩的，实用场景非常广泛，我们通过其使用方法及源码学习其设计思想
```

### Callbacks的参数


根据官方文档Callbacks的参数可能是一下其中的某一个或者多个组合:
* once               只会被fire一次
* memory             缓存最后一次fire的参数，再add回调函数时，内部将会直接进行fire并把缓存的值作为其参数 
* unique             同一个回调只允许被添加一次
* stopOnFalse        当回调函数返回值为false时停止执行后续回调

### 使用方法

1.  不传参数

```bash
var a = $.Callbacks()
a.add(v=>console.log(v))
a.fire(1)  //=> 1 
a.add(v=>console.log('second:'+v))
a.fire(2)  //=> 2  second:3
```

2. once 

```bash
var a = $.Callbacks('once')
a.add(v=>console.log(v))
a.fire(1)  //=> 1
a.add(v=>console.log('second:'+v)) //不会被添加
a.fire(2)  //只会被fire一次 之后将会失效
```

3. memory 

```bash
var a = $.Callbacks('memory')
a.add(v=>console.log(v))
a.fire(1)  //=> 1
a.add(v=>console.log('second:'+v)) //=>second:2
a.fire(2)  //2  second:3
```

4. once memory 

```bash
var a = $.Callbacks('once memory')
a.add(v=>console.log(v))
a.fire(1)  //=> 1
a.add(v=>console.log('second:'+v)) //=>second:2
a.fire(2)  //fire无效
```

5. unique  

```bash
var a = $.Callbacks('unique')
var fn = v=>console.log(v)
a.add(fn)
a.fire(1)  //=> 1
a.add(fn) //重复的无法添加
a.fire(2)  //=>2
```

6. stopOnFalse 

```bash
var a = $.Callbacks('stopOnFalse')
a.add(v=>{
    console.log(v)
    return false
})
a.add(v=>console.log('second:'+v))
a.fire(2)  //=>2
```

7. stopOnFalse memory

```bash
var a = $.Callbacks('stopOnFalse memory')
a.add(v=>{
    console.log(v)
    return false
})
a.add(v=>console.log('second:'+v))
a.fire(1)  //=>1
a.add(v=>console.log('third:'+v)) //add时也不会立即执行
```

### Callbacks源码分析
```bash
/*
     * Create a callback list using the following parameters:
     *
     *	options: an optional list of space-separated options that will change how
     *			the callback list behaves or a more traditional option object
     *
     * By default a callback list will act like an event callback list and can be
     * "fired" multiple times.
     *
     * Possible options:
     *
     *	once:			will ensure the callback list can only be fired once (like a Deferred)
     *                  只会被fire一次
     *
     *	memory:			will keep track of previous values and will call any callback added
     *					after the list has been fired right away with the latest "memorized"
     *					values (like a Deferred)
     *                  fire之后再add的回调将会执行参数为最后一次fire时传的参数
     * 
     *	unique:			will ensure a callback can only be added once (no duplicate in the list)
     *                  同一个回调只允许被添加一次
     *                  
     *	stopOnFalse:	interrupt callings when a callback returns false
     *                  当回调函数返回值为false时停止执行后续回调
     */
    jQuery.Callbacks = function (options) {

        // Convert options from String-formatted to Object-formatted if needed 解析参数 
        // (we check in cache first)
        options = typeof options === "string" ?
            (optionsCache[options] || createOptions(options)) :
            jQuery.extend({}, options);
        var // Last fire value (for non-forgettable lists)记录最后一次fire的参数
            memory,
            // Flag to know if list was already fired 是否fire过
            fired,
            // Flag to know if list is currently firing  是否正在执行回调队列的函数(个人觉得暂无实际用处)
            firing,
            // First callback to fire (used internally by add and fireWith) 队列应该开始的索引位置
            firingStart,
            // End of the loop when firing
            firingLength,
            // Index of currently firing callback (modified by remove if needed) 队列当前正在执行的函数的索引位置
            firingIndex, 
            // Actual callback list
            list = [],
            // Stack of fire calls for repeatable lists 其布尔值可以作为是否禁用的状态; 解决重复fired问题，但是看源码其该stack进行push时是依赖firing状态，感觉无用并不能解决死循环问题(如在add的回调函数中进行fire)
            stack = !options.once && [],
            // Fire callbacks
            fire = function (data) {
                memory = options.memory && data;
                fired = true;
                firingIndex = firingStart || 0;
                firingStart = 0;
                firingLength = list.length;
                firing = true;
                //遍历执行回调
                for (; list && firingIndex < firingLength; firingIndex++) {
                    if (list[firingIndex].apply(data[0], data[1]) === false && options.stopOnFalse) {
                        memory = false; // To prevent further calls using add
                        break;
                    }
                }
                firing = false;
                if (list) {
                    if (stack) {
                        if (stack.length) {
                            fire(stack.shift());
                        }
                    } else if (memory) {
                        //once momory模式时重置了list  因为上一次的list里的函数不会再调用，为了节省内存重新赋值空数组
                        list = [];
                    } else {
                        //once模式时 fire后将会禁用
                        self.disable();
                    }
                }
            },
            // Actual Callbacks object
            self = {
                // Add a callback or a collection of callbacks to the list
                add: function () {
                    if (list) {
                        // First, we save the current length 
                        var start = list.length;
                        (function add(args) { //匿名函数自执行遍历参数
                            jQuery.each(args, function (_, arg) {
                                var type = jQuery.type(arg);
                                if (type === "function") {
                                    if (!options.unique || !self.has(arg)) {  //判断当前是否是unique模式，如果是判断该回调函数是否已经存在，存在将不再进行push
                                        list.push(arg);
                                    }
                                } else if (arg && arg.length && type !== "string") { //当为数组或伪数组时递归调用add
                                    // Inspect recursively
                                    add(arg);
                                }
                            });
                        })(arguments);
                        // Do we need to add the callbacks to the
                        // current firing batch?
                        if (firing) {
                            firingLength = list.length;
                            // With memory, if we are not firing then
                            // we should call right away  
                        } else if (memory) {  // memory有值表明当前是memory模式并且fire过了，所以add时直接执行 并且队列执行的起始索引位置设为add前的队列的length
                            firingStart = start;
                            fire(memory);
                        }
                    }
                    return this;
                },
                // Remove a callback from the list
                remove: function () {
                    if (list) {
                        jQuery.each(arguments, function (_, arg) {
                            var index;
                            while ((index = jQuery.inArray(arg, list, index)) > -1) {
                                list.splice(index, 1);
                                // Handle firing indexes
                                if (firing) {
                                    if (index <= firingLength) {
                                        firingLength--;
                                    }
                                    if (index <= firingIndex) {
                                        firingIndex--;
                                    }
                                }
                            }
                        });
                    }
                    return this;
                },
                // Check if a given callback is in the list.判断回调是否已经在list了
                // If no argument is given, return whether or not list has callbacks attached.
                has: function (fn) {
                    return fn ? jQuery.inArray(fn, list) > -1 : !!(list && list.length);
                },
                // Remove all callbacks from the list
                empty: function () {
                    list = [];
                    firingLength = 0;
                    return this;
                },
                // Have the list do nothing anymore 使之禁用
                disable: function () {
                    list = stack = memory = undefined;
                    return this;
                },
                // Is it disabled?
                disabled: function () {
                    return !list;
                },
                // Lock the list in its current state
                lock: function () {
                    stack = undefined;
                    if (!memory) {
                        self.disable();
                    }
                    return this;
                },
                // Is it locked?
                locked: function () {
                    return !stack;
                },
                // Call all callbacks with the given context and arguments指定上下文和参数
                fireWith: function (context, args) {
                    if (list && (!fired || stack)) { //当未fire过或者stack的布尔值为true 如果stack为false表明要么是once模式，要么就lock了
                        args = args || [];
                        //args[0]为上下文,args[1]为参数
                        args = [context, args.slice ? args.slice() : args];
                        if (firing) {
                            stack.push(args);
                        } else {
                            fire(args);
                        }
                    }
                    return this;
                },
                // Call all the callbacks with the given arguments
                fire: function () {
                    self.fireWith(this, arguments);
                    return this;
                },
                // To know if the callbacks have already been called at least once
                fired: function () {
                    return !!fired;
                }
            };

        return self;
    };
```

###
