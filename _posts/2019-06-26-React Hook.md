---
title: React Hook - 不依赖生命周期的有状态组件
date: 2018-09-20 10:27:33
description: 对React Hook的一些测试，以及一些注意事项
categories:
 - React
tags: React
---

React Hook 是 React 16.8版本新增的特性，他的主要作用是可以让一些不利用复杂生命周期的组件，可以用Function Component的形式来写。大大简化了组件结构。

### useState

基础用法：
```javascript
  import * as React from 'react'

  const Example = () => {
    const [count, setCount] = React.useState(0)
    
    return (
      <div>
        <p>当前计数器数值： { count }</p>
        <button onClick={ () => setCount(count + 1) }></button>
      </div>
    )
  }
```

抽象用法：
```javascript

```