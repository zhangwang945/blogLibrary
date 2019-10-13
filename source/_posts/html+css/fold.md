---
title: 超出文本折叠效果
date: 2018-06-01 19:14:40
tags: html+css
categories: html+css
---

### 目的

```
使用html+css实现超出文本的折叠效果
```

### 实现原理
![示例图片](https://t1.picb.cc/uploads/2019/10/13/gUYASK.png "示例图片")
如上图,采用占位元素(限制区域大小保持一致))+ 挡板(继承内容的高度,向右浮动)
按钮也向右浮动，当内容高度不超过限制区域，按钮将浮动到最右面，被遮挡住,
当超出限制区域,则浮动到挡板的左下方,露出

### 最终效果
![示例图片](https://t1.picb.cc/uploads/2019/10/13/gUY7D7.png "示例图片")

### 代码
```bash
 <!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
    <style>
        .common-box {
            margin-right: 30px;
            height: 100px;
            width: 200px;
            line-height: 25px;
            border: 1px solid black;
            word-break: break-all;
            display: inline-block;
            vertical-align: top;
        }

        .box2 {
            position: relative;
            overflow: hidden;
        }

        .content-box {
            position: relative;
        }

        .box2 .content-box .fold-box {
            position: absolute;
            top: 0;
            /* left: 50%; */
            height: 100%;
            width: 200%;
        }

        .box2 .content-box .fold-box::before {
            content: '';
            float: right;
            height: 100%;
            width: 50%;
        }

        .box2 .content-box .fold-box .place-hold {
            height: 100px;
            width: 50%;
        }

        .box2 .content-box .fold-box .detail-btn {
            float: right;
            margin-top: -25px;
            background: linear-gradient(to right, rgba(255, 255, 255, 0) 10%, #ffffff 70%);
            /* padding-left: 30px; */
            width: 50%;
            box-sizing: border-box;
            text-align: right;
            color: #0084ff;
        }

        .box2 .content-box .fold-box .detail-btn span {
            background: #ffffff;
        }
    </style>
</head>

<body>
    未超出时效果：
    <div class="common-box box2">
        <div class="content-box">
            dfdf辅导费大fdsfsfdsf234234234234234234
            234dfsdfdfFdfd大幅度辅导费大幅度大幅度发到付
            <div class="fold-box">
                <div class="place-hold"></div>
                <div class="detail-btn"><span>显示详情</span></div>
            </div>
        </div>
    </div>
    超出时效果：
    <div class="common-box ">
        dfdf辅导费大幅度付地方大幅度发到付大幅地方大幅度发到付fdfdfdfdfdsfsfdsf234234234234234234
        234dfsdfdfFdfd大幅度辅导费大幅度发到付地方大幅度发到付</div>
    <div class="common-box box2">
        <div class="content-box">
            dfdf辅导费大幅度付地方大幅度发到付大幅地方大幅度发到付fdfdfdfdfdsfsfdsf234234234234234234
            234dfsdfdfFdfd大幅度辅导费大幅度发到付地方大幅度发到付
            <div class="fold-box">
                <div class="place-hold"></div>
                <div class="detail-btn"><span>显示详情</span></div>
            </div>
        </div>
    </div>

</body>

</html>

```