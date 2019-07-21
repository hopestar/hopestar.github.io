---
layout: post
title: "基于用户行为的视频防盗链方案"
date: 2019-07-21
description: 随着防盗链业务发展，公司的防盗链代码分布在客户端，CDN，调度等各个业务线，导致系统变得越来越复杂，排查问题、升级等都变得难以。我们需要一个系统，将防盗链策略跟各个系统解耦，只提供服务接口。而且只有在系统化服务的情况下，我们才能做出对数据实时性，一致性高的策略方案。如行为策略方案
img: timg.jpg 
tags: [tech, anti]
---
## 背景
随着防盗链业务发展，公司的防盗链代码分布在客户端，CDN，调度等各个业务线，导致系统变得越来越复杂，排查问题、升级等都变得难以。我们需要一个系统，将防盗链策略跟各个系统解耦，只提供服务接口。而且只有在系统化服务的情况下，我们才能做出对数据实时性，一致性高的策略方案。如行为策略方案。

## 简介
传统的防盗链是基于单次的http请求，根据url参数和refer等header头参数进行判断，基于url参数容易被拷贝，伪造等特点，导致传统防盗链有效性不高。
行为策略方案, 防盗链系统的核心策略。其核心理念就是通过一个全局的UserID，将用户分布在各个服务端的请求记录实时收集起来，依据**用户的行为**来判断该请求来自真实亦或盗链用户。
那么我们先来看一下真实用户和盗链用户在服务端的区别。

## 真实/盗链用户

一个内容厂商的app的打开后，一般正常的播放次序是：   
    A：开机画面   
    B：上报设备信息   
    C：加载播放列表   
    D：播放广告   
    E：播放视频   
一个真实的用户的访问行为，在我们内容厂商服务器观察到的是ABCDE，5种行为依次到达。
而盗链厂商则是绕过跟自身业务无法的业务，插入自身（即非原内容厂商）的业务，然后到达播放的步骤，所以在内容厂商服务器后端观察到的就只有E一种行为。
并且这种行为与客户端无关，无论是app,h5,pc等客户端，都具有以上的行为特征；

## 系统架构
![架构图](/assets/img/anti-theft-chain.png)
## 流程   
### 步骤：
1.业务方设置规则R；   
2.用户访问业务方服务器；   
3.业务方服务器上报用户执行行为到数据中心；数据中心按照splatid和userid等维度作为key存放在数据库；   
4.用户访问CDN；   
5.CDN每次播放或请求ts时候访问anti server, anti server查询数据库，判断是否符合业务方设置规则；   
6.返回CDN判断结果；   
7.返回用户结果；    
 
## 基本概念
- UserID
  用户的唯一标识，用户标识一位用户。由APP生成，存储在客户端，贯穿于所有核心流程的请求。
- 行为清空
  每次执行开机动作a,则清空key的先前记录，重新开始记录行为；
- 过期时间
  key值设置过期时间(由业务方设置)，防止不活跃用户占用太多内存。即默认天数过后重新记录行为；

## 架构demo
    设定行为规则为 a,b,c; 测试用例结果如下
    test1 : action = a
    {"status":200,"error":"success"}
    {"status":403,"error":"rule1 error"}
    [root@i-06csjvbm nginx]# sh test.sh  2
    test2 : action = a, b
    {"status":200,"error":"success"}
    {"status":200,"error":"success"}
    {"status":403,"error":"rule1 error"}
    [root@i-06csjvbm nginx]# sh test.sh  3
    test3 : action = a, b, c
    {"status":200,"error":"success"}
    {"status":200,"error":"success"}
    {"status":200,"error":"success"}
    {"status":200,"error":"success"}

## 各端接口
Userid: 用户ID; action：行为；splatid ：业务方ID；token：业务方标示；

后端（接入/上报端）api：
接入系统nginx日志 或提供统一的http接口
curl -i “localhost/api/upload?userid=123&action=a&splatid=1011&token=dianshijia&ip=127.0.0.1&device=xiaomi“

规则设置后台(二期做，需要存在sql里面)：
curl -i  “localhost/api/rule?rule1=abc&splatid=1011&token=dianshijia”

前端（CDN端）：
 ts请求:
curl -d "userid=123&splatid=1011" "localhost/cdn/get"

## 数据存储
#### redis cluster
- 高并发
- 高可用
- 数据结构丰富
- 与nginx结合紧密
- 有一定运维经验

## 监控
![监控](/assets/img/anti2.png)

## 柔性可用与优化
1.CDN设置超时时间(<=80ms），超时即默认验证通过，防止误杀；
2.anti server 设置默认值，流量暴涨/系统过载情况下默认返回验证通过，防止误杀；
3.网络大的延迟下，有可能行为a,b,c的到达次序不准确，可忽略行为次序，只判断步骤是否完全来判定，防止误杀；
4.CDN段设置缓存，对于相同的userid，设置5-60s左右的缓存。Anti server 不设置缓存。
5.结合User/IP频次控制，黑名单，白名单等多策略协同；

## 未来
- 结合机器学习进行数据挖掘，自动生成拦截规则。

## 小结
- 核心概念：基于用户行为判断用户身份
- 功能：区分盗链正常用户；
- 架构：数据上报，规则设置，行为判断

随着人工智能和数据挖掘是未来软件发展的趋势，基于用户行为处理是基于数据挖掘在防盗链方向的应用，很好的规避了传统防盗链的缺点，达到了更加准确的识别效果。