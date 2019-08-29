---
title: RxJs与axios的合并使用
date: 2019-05-09 15:41:48
description: 日常工作中碰到的rxjs与axios请求库的联合使用问题
categories:
 - 日常工作
tags: 日常工作,经验
---

在vue架构中，axios确实是很好用的请求库，大多数时候，我们都在使用它，反而很少去使用fetch库。但是axios的处理方式很特殊，通过生成的axiosStatic对象的get、post、put、delete方法调用。一旦调用，便只能通过cencelToken的回调方法调用取消了。

RxJs给我们在流map提供了很多操作符，其中switchMap比较特殊，虽然switchMap一定要等到上一个订阅完成之后，才能开始订阅下一个Observable，如果中途不断会有新的数据产生，那么就会立刻放弃订阅，转而订阅新的Observable。

当我们在swichMap当中执行请求的时候，难免会碰到前一个请求已经发出去了，但是我们没有订阅，从官网文档上来看，说是会取消之前的请求发送？。但是我尝试失败了。无形当中的DDOS就在此产生了。

后续在值得传递过程中，发现switchMap中其实还在接收值，只不过在前一个Observable还未complete的情况下，是不会订阅的。但是axios在create之后的使用无论是get、post、put、delete等方法都会立即发送请求出去，所以就有一个想法，能否使axios本身的请求过程变成一个流的形式，在被订阅时发出请求，取消订阅时也取消请求。

至此封装了一个应对axios的Observable库: [axios-rx-observable](https://github.com/lintakejs/axios-rx-observable)