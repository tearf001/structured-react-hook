## structured-react-hook

> [Document](./README_EN.md)

面向企业级的结构化的 React 状态管理框架

完全基于 React Hook 的原生方案

> 在线 [Demo](https://codesandbox.io/s/strange-field-l5ogh) 展示了一个较为复杂的 TodoApp

### 安装

```
yarn add structured-react-hook
or
npm install structured-react-hook
```

### 用一个基础示例快速开始

这里以编写一个最基本例子, 点击按钮, 显示 loading, 调用服务端, 移除 loading, 显示服务端返回的结果

```js
import React from 'react'
import createStore from 'structured-react-hook'

const storeConfig = {
    name:'testStore',
    initState:{
        loading: false,
        content: '尚未加载任何数据'
    },
    controller:{
        async onButtonClick(){
            this.rc.setLoading(true)
            this.rc.setContent('数据加载中...')
            // 此处请求服务端
            const res = await api.get('api/commit')
            this.rc.setLoading(false)
            this.rc.setContent(res.data.content)
        }
    }
}
const useStore = createStore(storeConfig)
function Example(){
    const store = useStore()
    return(
        <div>
            <p>{store.state.content}</p>
            <button onClick={store.controller.onButtonClick}></button>
        </div>
    )
}

```

### 一个更复杂的场景

```js
import React from 'react'
import createStore from 'structured-react-hook'

const storeConfig = {
    name:'testStore',
    initState:{
        loading: false,
        content: '尚未加载任何数据'
    },
    service:{
        async query(){
            const res = await api.get('api/query');
            return res.data
        }
    },
    controller:{
        async onButtonClick(){
            this.rc.setState({
                loading:true,
                content:'数据加载中...'
            })
            // 此处请求服务端
            const data = await this.service.query()
            this.rc.setState({
                loading:false,
                content: data
            })
        }
    }
}
const useStore = createStore(storeConfig)
function Example(){
    const store = useStore()
    return(
        <div>
            <p>{store.state.content}</p>
            <button onClick={store.controller.onButtonClick}></button>
        </div>
    )
}

```

## 核心概念

### 严格的单向数据流

structured-react-hook 采用严格的单向数据流架构, 遵循

View → Controller[→ Service →]→ Controller → View 的数据控制流向, 包括以下规则


- View 无法获取 Store 内定义的 Service
- Controller 无法调用到其他 Controller, 除非通过 membrane 调用需要重载的同名 Controller
- Service 无法调用 setState 来修改 View 的 state 触发 render

> 关于 membrane 见后面的 高级 API 部分

### 遵循 flux 的多 store 设计

和 redux 倡导的单一 store 理念不同, 再采用原生的 useReducer 来控制状态, 天然就遵守了 flux 的多 store 的设计思路, 

并且通过大量的工程实践, 单一 store 也是一个看起来美好但不可行的方案, 主要是因为需求业务的不可知性, 并且当业务变得庞大且复杂,

单一 store 的边际效应会急剧下降, 甚至会反过来增加维护成本.

### 驱动 React 的秘密

和 Redux 不同, structured-react-hook 不需要依赖类似 react-redux 这样的驱动库来连接 react 和 redux, 其秘密在于, structured-react-hook 内部使用了 useReducer hook, 利用 react 原生 api 的能力管理状态

### 何为 StoreConfig

store 是一个普通的 js 对象包含以下属性

```js

const storeConfig = {
    name:'', // 必选 用于区分 store 的唯一性
    initState:{} // 必选 useReducer 的第一个参数
    initState:[
        {},()=>{}
    ] // initState 增强模式, 数组的第一个参数依然是 initState, 第二个参数是 useReducer 的 init 函数, 用于延迟 initState 的计算
    ref:{} // 一个对象, structured-react-hook 使用 useRef hook 遍历这个对象, 生成对应的 ref
    view:{} // 管理动态 jsx, 可以编写 render函数
    service:{} // 被提取出来的 controller 逻辑, 当 controller 变得复杂的时候, 你可能需要它
    controller:{} // store 的控制器, 控制逻辑的运行和 UI 状态的设定, controller 必须以  on 开头 + 名词 + 动词 来命名
}
```
### Store 上下文

structured-react-hook 为所有 Store 提供了统一的上下文, 具体示例见后续 createStoreContext API, 这里提到的是每个 Store 独立的上下文. Store 上下文为 Store 内部的方法, controller view service 提供了方位状态和彼此调用的能力, 上下文挂载在 this 上, 你可以在 view controller 和 service 中调用到他, 上下文包括

- rc
- view
- service
- controller
- state
- refs
- name
- context(所有 store 共享的上下文, 一个全局 store)
- super(membrane 模式下)
- props(membrane 模式下)

### 通过 rc 控制 react state 的秘密

你可能会好奇, structured-react-hook 内部使用了 useReducer, 为什么没有可用的 dispatch 或者 action 以及为什么不需要书写 reducer? 其秘密在于 structured-react-hook 在你声明 initState 的时候就自动生成了对应的操作方法, 并挂载到 Store 上下文 this.rc 上了, rc 是 reducer case 的缩写, 因此你不需要像 redux 那样编写繁琐的 action type 和 reducer



### 场景化示例

#### 使用 rc 驱动 React 函数组件

```js
import React from 'react';
import createStore from 'structured-react-hook';

const storeConfig = {
    name:'calc',
    initState:{
        count:0
    },
    controller:{
        onAddButtonClick(){
            this.rc.setCount(this.state.count++)
        },
        onSubButtonClick(){
            this.rc.setCount(this.state.count--)
        }
    }
}

const useStore = createStore(storeConfig)

function App(){
    const store = useStore()

    return (
        <div>{store.state.count}</div>
        <button onClick={store.controller.onAddButtonClick}>+ 1</button>
        <button onClick={store.controller.onSubButtonClick}>- 1</button>
    )
}

```

#### 使用 this.rc.setState 批量修改状态

```js
import React from 'react';
import createStore from 'structured-react-hook';

const storeConfig = {
    name:'calc',
    initState:{
        count:0,
        count1:1,
        count2:2,
    },
    controller:{
        onAddButtonClick(){
            this.rc.setState({
                count:this.state.count++,
                count1:this.state.count1++,
                count2:this.state.count2++,
            })
        },
        onSubButtonClick(){
            this.rc.setState({
                count:this.state.count--,
                count1:this.state.count1--,
                count2:this.state.count2--
            })
        }
    }
}

const useStore = createStore(storeConfig)

function App(){
    const store = useStore()

    return (
        <div>{store.state.count}</div>
        <div>{store.state.count1}</div>
        <div>{store.state.count2}</div>
        <button onClick={store.controller.onAddButtonClick}>+ 1</button>
        <button onClick={store.controller.onSubButtonClick}>- 1</button>
    )
}

```

#### 用 ref 代替 classComponent 中的 this, 实现一些非 UI 消费状态的声明

当你的 storeConfig 中存在一些需要保存下来的值, 但是又不需要吐到 UI 中, 就可以考虑使用 ref

> 更多场景化的示例可见测试用例 test 目录下的测试用例

## Membrane 模式

membrane 模式是我们在企业级应用开发的场景下提炼出来的用于延长代码生命周期的武器, 简单的说他可以保护你代码中最重要的部分

让我们用一个实际场景来理解 membrane 

在实际开发中, 我们经常会遇到的一个问题是针对不同渠道在业务流程处理上有所差异, 但随着时间推移, 这种差异会逐渐侵蚀污染我们
代码的可复用性.

例如

```js

function main(){
    if(channel === 'h5'){
        // todo sth
    }
    todo sth
}

function sub(){
    if(channel === 'h5'){
        // todo sth
    }
    todo sth
}

```

这种逻辑分支大量存在于真实的软件工程内部, 当我们需要快速复用 main 和 sub 函数内的逻辑, 就需要进行重构, 但真实的场景远比这个复杂.
我们通常没有足够的时间去重构, 于是就只能...

```js

function main(){
    if(channel === 'h5'){
        // todo sth
        if(platform === 'xxx'){
            ....
        }
    }
    todo sth
}

function sub(){
    if(channel === 'h5'){
        // todo sth
    }
    todo sth
}

```

解决 if 可能有很多种方法, 但目前为止并没有一种方法能够有效的将这种分支逻辑归一到一起, 即便你勤于重构但最终依然会散落在整个工程的各个部分
为了解决这个问题, 我们创造了 membrane, 一种可插拔的分支逻辑的归一模式

```js
// 这段代码用于解释什么是 membrane , 为此做了精简, 实际使用会稍有不同
const storeConfig = {
    main(){
        //todo sth
    },
    sub(){
        // todo sth
    }
    membrane:{
        main(){
            if(channel === 'h5'){
                // todo sth
                if(platform === 'xx'){
                    // todo sth
                }
            }
            this.super.main()
        }
        sub(){
            if(channel === 'h5'){
                // todo sth
            }
            // todo sth
        }

    }
}

```

如果要解释 membrane 背后的设计, 恐怕得占据不少篇幅, 关于这部分可以见后续 docs 内的内容, 在这里我简单概括下
membrane 的灵感来自于细胞膜的生物结构, 同时借鉴了 hook 可插拔的设计还有面向对象中的继承和函数重载. 下面是一个真实的用例

```js
import React from 'react'
import createStore from 'structured-react-hook'

const storeConfig = {
    initState:{
        name: jacky
    },
    controller:{
        onButtonClick(){
            this.rc.setName('jacky in main')
        }
    },
    membrane:{
        controller:{
            onButtonClick(){
                if(this.props.platform === 'h5'){
                    this.rc.setName('jacky in h5)
                }
            }
        }
    }
}

const useStore = createStore(storeConfig)

function App(){
    const store = useStore('h5')
    return(
        <div>{this.state.name}</div> // jacky in h5
    )
}

function App(){
    const store = useStore()
    return(
        <div>{this.state.name}</div> // jacky in main
    )
}

```

如果你要复用这个 storeConfig, 你只需要排除 membrane 即可, 这就是 membrane 的可插拔.


