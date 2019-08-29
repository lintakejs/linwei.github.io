---
title: 请不要轻易使用switchMap
date: 2019-07-20 15:30:23
description: mergeMap()、switchMap()、concatMap()、exhaustMap()的区别
categories:
 - rxjs
tags: rxjs
---

## 关于为什么会出现诸多map操作符
让我们从正常的情况开始
```javascript
  const namesObservable = of('li', 'wang')
  namesObservable.pipe(
    .map((name) => `${name} 真的是太强大了！`)
  ).subscribe((result) => console.log(result))

  // console
  // li 真的是太强大了！
  // wang 真的是太强大了！
```
这样是正常的。但是如果我们需要将name去远端请求，或者去其他流处理之后呢？

像是这样:
```javascript
  const http = {
    getAwesomeMessagesObservable (name) {
      return of(`${name} 真的是太强大了`).pipe(dalay(1000))
    }
  }
```
然后将他们串联
```javascript
  namesObservable.pipe(
    map((name) => http.getAwesomeMessagesObservable(name))
  ).subscribe((result) => console.log(result))
  // console
  // [object object]
  // [object object]
```
这时候map当中返回的是相对应的observable，而不是值。而从里边取值唯一的形式则是订阅他。
```javascript
  namesObservable.pipe(
    map((name) => http.getAwesomeMessagesObservable(name))
  ).subscribe((result) => {
    result.subscribe(res => console.log(res))
  })
```
在这种情况下，只不过是合并了订阅罢了，层级数增加了，这不是我们想要的结果，我们希望更加平行的流式处理。

## 4种形式的扁平策略

1. Merge形式，实际上他并没有做什么事情，只不过在不断接收与订阅。

2. Switch形式，当有新的observable到达，会放弃监听最后来自于map最后的observable，转而订阅新的observable。

3. Concat形式，map当中的返回的observable形成队列，只有当上一个订阅完成后，才会监听下一个订阅。

4. Exhaust形式，是Swith模式的对立，如果有订阅的observable未完成，则会放弃新进的observable。

每一种策略其实对应着一种操作符。

1. mergeMap()

根据Merge形式的策略，mergeMap其实并没有做什么，只不过返回了对于每一个map返回的值形成observable，并且订阅拿值。我们可以直接改造上边的例子。

```javascript
  namesObservable.pipe(
    mergeMap((name) => http.getAwesomeMessagesObservable(name))
  ).subscribe((result) => console.log(result))
  // console
  // ...1s
  // li 真的是太强大了
  // wang 真的是太强大了
```
在上边的例子当中，我们忽略了一个重要的问题，当远端服务器的请求返回是不可控的，造成的是数据流的响应可能是错乱的。因为每个新的流数据都会引发订阅响应。

有一个商品列表，点击会弹出商品详情，这时候用户如果点击了多个商品，最后可能会造成的商品信息不对等的不可控场面。所以mergeMap并不适合用在需要很强执行顺序关系的场景下。例如：某区域队长移除/添加自己队伍中队员，每一次的移除/添加互不影响。

```javascript
  areaTlRemoveOrAdd.pipe(
    mergeMap((teamMember) => from(fetch('teamRemoveOrAdd')))
  ).subscribe((result) => ...)
```

2. switchMap()

根据Switch策略，新的observable到来，就会放弃旧的订阅。我们改造最上边的模拟远程返回的例子。

```javascript
  namesObservable.pipe(
    switchMap((name) => http.getAwesomeMessagesObservable(name))
  ).subscribe((result) => console.log(result))
  // console
  // ...1s
  // wang 真的是太强大了
```

在上边的例子中，之前的流数据输入被放弃订阅了。这意味着，如果有远端操作，无论返回状态时间如何，都不会被订阅，他只订阅最新的一个。

有一场景，用户点击商品详情查看某商品详细信息，用户连续点击，则原本的请求会被取消订阅，然后不断产生新的请求，实际上是对服务器的DDOS。switchMap太强调统一类型操作被调用，先前的请求应该被中止订阅了，每一次的流数据变更，先前还未返回的请求都将被中止订阅，但实际上，请求还是发出了，所以switchMap更适合用在
同类型操作应该被覆盖执行的场景，例如：某搜索栏，当用户input后，执行远端搜索，同类型应该需要被覆盖，而搜索最新的流数据。

```javascript
  fromEvent(document.getElementById('serach'), 'input').pipe(
    switchMap((searchValue) => from(fetch('searchProduct')))
  ).subscribe((result) => console.log(result))
```

3. concatMap()

根据concat策略，新的observable到来，一定要等到旧的订阅完成之后，才会继续新的订阅。我们继续改造最上边的模拟远程返回的例子

```javascript
  namesObservable.pipe(
    concatMap((name) => http.getAwesomeMessagesObservable(name))
  ).subscribe((result) => console.log(result))
  // console
  // ...1s
  // li 真的是太强大了
  // ...1s
  // wang 真的是太强大了
```
在上边的例子中，看起来像是个队列，只有前一个完成了，才执行下一个，这是对一个操作流程的硬性掌控。是相对mergeMap更加安全的做法，他的返回序列是相对应的，能够解决mergeMap例子当中产生的问题。

4. exhaustMap

根据Exhaust形式，旧的订阅还未完成，新的订阅就不会生效，我们最后在改造一次最上边的模拟远程返回的例子。

```javascript
  namesObservable.pipe(
    exhaustMap((name) => http.getAwesomeMessagesObservable(name))
  ).subscribe((result) => console.log(result))
  // console
  // ...1s
  // li 真的是太强大了
```
在上边的例子当中，在第一个输入的请求还未返回结果时，是不会触发第二个流数据进入新的请求订阅当中的。所以exhaustMap在类似于登录，或者防止重复多次点击的地方会有妙用。

## 总结

* 当操作不能被中止/忽略，且必须保证执行顺序的情况下，使用 concatMap；（这也是一个保守的选择，因为其总会按照预期运行）
* 当操作不能被中止/忽略，但执行顺序并不重要的情况下，使用 mergeMap；
* 当处理读操作时，且当另一个相同类型的操作被调用时，先前的请求应当被中止的情况下，使用 switchMap；
* 当相同类型的操作应当被忽略的情况下，使用 exhaustMap。

附各大map的弹珠图：

  mergeMap:

  ![mergeMap](https://i.stack.imgur.com/xd5cd.png)

  concatMap:

  ![concatMap](https://i.stack.imgur.com/EN4Uw.png)

  switchMap:

  ![switchMap](https://i.stack.imgur.com/FKTG5.png)

  exhaustMap

  ![exhaustMap](https://rxjs.dev/assets/images/marble-diagrams/exhaustMap.png)