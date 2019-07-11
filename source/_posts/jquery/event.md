---
title: event源码分析
date: 2018-07-04 22:16:24
tags: jquery
categories: jquery
---

### 事件绑定的与触发的大致原理

事件绑定过程(假设第一次绑定)
>1. 查询data_priv缓存是否有该dom对应事件缓存,有则直接获取，没有则添加缓存,缓存的key值(钥匙)会绑定到dom元素的属性[data_priv.expando]上,当再次添加事件时，通过该属性知道其缓存的key值从而读取缓存
>2. 为新缓存添加events空对象(同一元素的同一事件只会第一次执行)
>3. 生成的eventhandle函数事件触发的回调(该回调主要用于dispatch触发事件队列),并将该dom作为elem属性值,将该函数也添加到缓存上(同一元素的同一事件只会第一次执行)
>4. 生成handleObj存储一些该事件的元信息
>5. 为events对应的事件初始化handlers队列、代理函数最后位置(同一元素的同一事件只会第一次执行)
>6. eventhandle绑定到dom对应的事件句柄上(同一元素的同一事件只会第一次执行)
>7. 将存有相关信息的handleOboj添加到handlers队列中去

事件触发

1. dispatch(触发事件)
>1. 通过dom的[data_priv.expando]属性获取data_priv缓存的key，从而取到对应事件的hanlers队列
>2. 筛选出符合触发条件的对象重新分类 默认事件代理的对象在前，普通在后
>3. 遍历执行所有的符合条件的处理函数

2. trigger(模仿原生事件需手动触发，常用于触发自定义事件)
>1. 收集该dom冒泡路径上的dom 按顺序添加到数组中
>2. 遍历冒泡顺序的dom集合 查询data_priv缓存是否存在该事件的缓存信息，存在则调用handle事件触发函数(绑定事件时的eventHandle 事件触发函数)
### event相关源码分析

```bash
jQuery.event = {
       //用于绑定事件
       add: function (elem, types, handler, data, selector) {
            debugger
            var handleObjIn, eventHandle, tmp,
                events, t, handleObj,
                special, handlers, type, namespaces, origType,
                //获取该element的事件对象缓存 通过检测elem[data_priv.expando] 是否有值判断是否缓存过，有则获取缓存对象，没有则创建缓存对象并返回空对象{}
                elemData = data_priv.get(elem);

            // Don^t attach events to noData or text/comment nodes (but allow plain objects)
            if (!elemData) {
                return;
            }

            // Caller can pass in an object of custom data in lieu of the handler
            if (handler.handler) {
                handleObjIn = handler;
                handler = handleObjIn.handler;
                selector = handleObjIn.selector;
            }

            // Make sure that the handler has a unique ID, used to find/remove it later 
            //回调函数挂载唯一 id
            if (!handler.guid) {
                handler.guid = jQuery.guid++;
            }

            // Init the element^s event structure and main handler, if this is the first
            //检测该元素缓存没有events时添加个空events对象      elemData={events:{}}
            if (!(events = elemData.events)) {
                events = elemData.events = {};
            }
            if (!(eventHandle = elemData.handle)) {
                //elemData={events:{},handle:eventHandle,}  
                //该函数会绑定到原生dom事件句柄上
                eventHandle = elemData.handle = function (e) {
                    debugger
                    // Discard the second event of a jQuery.event.trigger() and
                    // when an event is called after a page has unloaded
                    //判断是否trigger过并且triggered事件类型是否与原生的类型一致
                    return typeof jQuery !== core_strundefined && (!e || jQuery.event.triggered !== e.type) ?
                        jQuery.event.dispatch.apply(eventHandle.elem, arguments) :
                        undefined;
                };
                // Add elem as a property of the handle fn to prevent a memory leak with IE non-native events
                //将dom挂载到其事件处理函数上
                eventHandle.elem = elem;
            }

            // Handle multiple events separated by a space
            //处理多个事件字符串 'click mouseover'  返回数组[click mouseover]
            types = (types || "").match(core_rnotwhite) || [""];
            t = types.length;
            while (t--) {
                tmp = rtypenamespace.exec(types[t]) || [];
                type = origType = tmp[1]; //事件
                namespaces = (tmp[2] || "").split(".").sort();

                // There *must* be a type, no attaching namespace-only handlers
                if (!type) {
                    continue;
                }

                // If event changes its type, use the special event handlers for the changed type
                special = jQuery.event.special[type] || {};

                // If selector defined, determine special event api type, otherwise given type
                type = (selector ? special.delegateType : special.bindType) || type;

                // Update special based on newly reset type
                special = jQuery.event.special[type] || {};

                // handleObj is passed to all event handlers
                handleObj = jQuery.extend({
                    type: type,
                    origType: origType,
                    data: data,
                    handler: handler,
                    guid: handler.guid,
                    selector: selector,
                    needsContext: selector && jQuery.expr.match.needsContext.test(selector),
                    namespace: namespaces.join(".")
                }, handleObjIn);

                // Init the event handler queue if we^re the first
                //初始化缓存的对应事件的函数队列 如elemData={events:{click:[]},handle:eventHandle,}
                if (!(handlers = events[type])) {
                    handlers = events[type] = []; //对应事件的函数队列
                    handlers.delegateCount = 0; //事件代理handle在函数队列中的位置

                    // Only use addEventListener if the special events handler returns false
                    if (!special.setup || special.setup.call(elem, data, namespaces, eventHandle) === false) {
                        if (elem.addEventListener) {
                            //dom绑定事件触发函数
                            elem.addEventListener(type, eventHandle, false);
                        }
                    }
                }

                if (special.add) {
                    special.add.call(elem, handleObj);

                    if (!handleObj.handler.guid) {
                        handleObj.handler.guid = handler.guid;
                    }
                }

                // Add to the element^s handler list, delegates in front
                //如果有值表明是事件委托，在函数队列中插入
                if (selector) {
                    handlers.splice(handlers.delegateCount++, 0, handleObj);
                } else {
                    handlers.push(handleObj);
                }

                // Keep track of which events have ever been used, for event optimization
                jQuery.event.global[type] = true;
            }

            // Nullify elem to prevent memory leaks in IE
            elem = null;
        },
        //原生事件时一般调用函数用于通知处理该dom的对应的事件的所有处理函数
        dispatch: function (event) {

            // Make a writable jQuery.Event from the native event object
            event = jQuery.event.fix(event);

            var i, j, ret, matched, handleObj,
                handlerQueue = [],
                args = core_slice.call(arguments),
                //事件函数队列
                handlers = (data_priv.get(this, "events") || {})[event.type] || [],
                special = jQuery.event.special[event.type] || {};

            // Use the fix-ed jQuery.Event rather than the (read-only) native event
            args[0] = event;
            event.delegateTarget = this;

            // Call the preDispatch hook for the mapped type, and let it bail if desired
            if (special.preDispatch && special.preDispatch.call(this, event) === false) {
                return;
            }

            // Determine handlers
            //事件队列中筛选出符合触发条件的对象 [{el,handlers},{el,handlers}]
            handlerQueue = jQuery.event.handlers.call(this, event, handlers);

            // Run delegates first; they may want to stop propagation beneath us
            i = 0;
            //判断是否阻止默认行为 遍历筛选出的队列
            while ((matched = handlerQueue[i++]) && !event.isPropagationStopped()) {
                event.currentTarget = matched.elem;

                j = 0;
                //判断函数事件队列并逐个触发
                while ((handleObj = matched.handlers[j++]) && !event.isImmediatePropagationStopped()) {

                    // Triggered event must either 1) have no namespace, or
                    // 2) have namespace(s) a subset or equal to those in the bound event (both can have no namespace).
                    if (!event.namespace_re || event.namespace_re.test(handleObj.namespace)) {

                        event.handleObj = handleObj;
                        event.data = handleObj.data;
                        //触发事件函数
                        ret = ((jQuery.event.special[handleObj.origType] || {}).handle || handleObj.handler)
                            .apply(matched.elem, args);

                        if (ret !== undefined) {
                            if ((event.result = ret) === false) {
                                event.preventDefault();
                                event.stopPropagation();
                            }
                        }
                    }
                }
            }

            // Call the postDispatch hook for the mapped type
            if (special.postDispatch) {
                special.postDispatch.call(this, event);
            }

            return event.result;
        },
        //触发函数，可以手动触发 主要用于自定义事件的触发
        trigger: function (event, data, elem, onlyHandlers) {
            debugger
            var i, cur, tmp, bubbleType, ontype, handle, special,
                eventPath = [elem || document],
                type = core_hasOwn.call(event, "type") ? event.type : event,
                namespaces = core_hasOwn.call(event, "namespace") ? event.namespace.split(".") : [];

            cur = tmp = elem = elem || document;

            // Don^t do events on text and comment nodes
            if (elem.nodeType === 3 || elem.nodeType === 8) {
                return;
            }

            // focus/blur morphs to focusin/out; ensure we^re not firing them right now
            if (rfocusMorph.test(type + jQuery.event.triggered)) {
                return;
            }

            if (type.indexOf(".") >= 0) {
                // Namespaced trigger; create a regexp to match event type in handle()
                namespaces = type.split(".");
                type = namespaces.shift();
                namespaces.sort();
            }
            ontype = type.indexOf(":") < 0 && "on" + type;

            // Caller can pass in a jQuery.Event object, Object, or just an event type string
            event = event[jQuery.expando] ?
                event :
                new jQuery.Event(type, typeof event === "object" && event);

            // Trigger bitmask: & 1 for native handlers; & 2 for jQuery (always true)
            event.isTrigger = onlyHandlers ? 2 : 3;
            event.namespace = namespaces.join(".");
            event.namespace_re = event.namespace ?
                new RegExp("(^|\\.)" + namespaces.join("\\.(?:.*\\.|)") + "(\\.|$)") :
                null;

            // Clean up the event in case it is being reused
            event.result = undefined;
            if (!event.target) {
                event.target = elem;
            }

            // Clone any incoming data and prepend the event, creating the handler arg list
            data = data == null ? [event] :
                jQuery.makeArray(data, [event]);

            // Allow special events to draw outside the lines
            special = jQuery.event.special[type] || {};
            if (!onlyHandlers && special.trigger && special.trigger.apply(elem, data) === false) {
                return;
            }

            // Determine event propagation path in advance, per W3C events spec (#9951)
            // Bubble up to document, then to window; watch for a global ownerDocument var (#9724)
            //确定向上冒泡路径把相关dom 按顺序添加数组中
            if (!onlyHandlers && !special.noBubble && !jQuery.isWindow(elem)) {

                bubbleType = special.delegateType || type;
                if (!rfocusMorph.test(bubbleType + type)) {
                    cur = cur.parentNode;
                }
                for (; cur; cur = cur.parentNode) {
                    eventPath.push(cur);
                    tmp = cur;
                }

                // Only add window if we got to document (e.g., not plain obj or detached DOM)
                if (tmp === (elem.ownerDocument || document)) {
                    eventPath.push(tmp.defaultView || tmp.parentWindow || window);
                }
            }

            // Fire handlers on the event path
            i = 0;
            //沿着路径的ddom顺序查找是否含有对应事件的触发函数handle以及事件队列handlers
            while ((cur = eventPath[i++]) && !event.isPropagationStopped()) {

                event.type = i > 1 ?
                    bubbleType :
                    special.bindType || type;

                // jQuery handler
                handle = (data_priv.get(cur, "events") || {})[event.type] && data_priv.get(cur, "handle");
                if (handle) {
                    handle.apply(cur, data);
                }

                // Native handler
                handle = ontype && cur[ontype];
                if (handle && jQuery.acceptData(cur) && handle.apply && handle.apply(cur, data) === false) {
                    event.preventDefault();
                }
            }
            event.type = type;

            // If nobody prevented the default action, do it now
            if (!onlyHandlers && !event.isDefaultPrevented()) {

                if ((!special._default || special._default.apply(eventPath.pop(), data) === false) &&
                    jQuery.acceptData(elem)) {

                    // Call a native DOM method on the target with the same name name as the event.
                    // Don^t do default actions on window, that^s where global variables be (#6170)
                    if (ontype && jQuery.isFunction(elem[type]) && !jQuery.isWindow(elem)) {

                        // Don^t re-trigger an onFOO event when we call its FOO() method
                        tmp = elem[ontype];

                        if (tmp) {
                            elem[ontype] = null;
                        }

                        // Prevent re-triggering of the same event, since we already bubbled it above
                        jQuery.event.triggered = type;
                        elem[type]();
                        jQuery.event.triggered = undefined;

                        if (tmp) {
                            elem[ontype] = tmp;
                        }
                    }
                }
            }

            return event.result;
        },
}
```

```bash

```