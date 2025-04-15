## 多用服务端组件

- 我们引入 `day.js` 的 SidebarNoteList 组件使用的是服务端渲染，这意味着 `day.js` 的代码并不会被打包到客户端的 bundle 中。

- 但如果你现在在 SidebarNoteList 组件的顶部添加 `'use client'`，声明为客户端组件，你会发现立刻就多了 day.js

这就是使用 React Server Compoent 的好处之一，服务端组件的代码不会打包到客户端的 bundle 中：

## 使用指南

我们声明了一个 `Sidebar` 组件用于实现侧边栏，其中有一个子组件 `SidebarNoteList` 用于实现侧边栏的笔记列表部分，针对每一条笔记，我们抽离了一个 `SidebarNoteItem` 组件来实现，在 `SidebarNoteItem` 中，我们又抽离了一个名为 `SidebarNoteItemContent` 的客户端组件用于实现展开和收回功能，然后我们在 `SidebarNoteItem` 这个服务端组件中将笔记的标题和时间这段 JSX 作为
`children` 传递给 `SidebarNoteItemContent`。

因为这段功能的实现涉及到我们开发 Next.js 项目常用的服务端组件和客户端组件导入：

1.  **服务端组件可以导入客户端组件，但客户端组件并不能导入服务端组件**
2.  **从服务端组件到客户端组件传递的数据需要可序列化**，以刚才的例子为例：

```js
// components/SidebarNoteItem.js

export default function SidebarNoteItem({ noteId, note }) {
  // ...
  return (
    <SidebarNoteItemContent
      id={noteId}
      title={note.title}
      fun={() => {}}
      expandedChildren={
        <p className="sidebar-note-excerpt">
          {content.substring(0, 20) || <i>(No content)</i>}
        </p>
      }
    >
      <header className="sidebar-note-header">
        <strong>{title}</strong>
        <small>{dayjs(updateTime).format('YYYY-MM-DD hh:mm:ss')}</small>
      </header>
    </SidebarNoteItemContent>
  )
}
```

3. **但你可以将服务端组件以 props 的形式传给客户端组件**，其实刚才的实现里就展现了两种传递服务端组件的形式：

```js
// components/SidebarNoteItem.js

export default function SidebarNoteItem({ noteId, note }) {
  const { title, content = '', updateTime } = note
  return (
    <SidebarNoteItemContent
      id={noteId}
      title={note.title}
      // 第一种方式
      expandedChildren={
        <p className="sidebar-note-excerpt">
          {content.substring(0, 20) || <i>(No content)</i>}
        </p>
      }
    >
      // 第二种方式
      <header className="sidebar-note-header">
        <strong>{title}</strong>
        <small>{dayjs(updateTime).format('YYYY-MM-DD hh:mm:ss')}</small>
      </header>
    </SidebarNoteItemContent>
  )
}
```

## 服务端组件特性

```js
// components/SidebarNoteItem.js
import dayjs from 'dayjs'

import SidebarNoteItemContent from '@/components/SidebarNoteItemContent'

export default function SidebarNoteItem({ noteId, note }) {
  const { title, content = '', updateTime } = note
  return (
    <SidebarNoteItemContent
      id={noteId}
      title={note.title}
      expandedChildren={
        <p className="sidebar-note-excerpt">
          {content.substring(0, 20) || <i>(No content)</i>}
        </p>
      }
    >
      <header className="sidebar-note-header">
        <strong>{title}</strong>
        <small>{dayjs(updateTime).format('YYYY-MM-DD hh:mm:ss')}</small>
      </header>
    </SidebarNoteItemContent>
  )
}
```

，`SidebarNoteItem` 是一个服务端组件，在这个组件中我们引入了 dayjs 这个库，然而我们却是在 `SidebarNoteItemContent` 这个客户端组件中使用的 dayjs。请问最终客户端的 bundle 中是否会打包 dayjs 这个库？

所以答案是不会。**在服务端组件中使用 JSX 作为传递给客户端组件的 prop，JSX 会先进行服务端组件渲染，再发送到客户端组件中**。

**“尽可能将客户端组件在组件树中下移”**，这里就是一个很好的例子。我们本可以直接把 `SidebarNoteItem` 声明为客户端组件，然后直接在这个组件里全部实现，但是却抽离了一个名为 `SidebarNoteItemContent` 的客户端组件用于实现展开和收回功能。

`SidebarNoteItemContent` 的内容原本是 `SidebarNoteList` 的子组件，现在却是 `SidebarNoteItem` 的子组件。虽然在组件树中的位置下移了，但我们却因此避免了 dayjs 这个库被打包到客户端 bundle 中。在开发的时候，应该尽可能缩减客户端组件的范围。

Suspense 的效果就是允许你推迟渲染某些内容，直到满足某些条件（例如数据加载完毕）。在开发 Next.js 项目的时候，有数据加载的地方多考虑是否可以使用 `Suspense` 或者 `loading.js`带来更好的体验。

## RSC Payload

如果你用 Chrome 查看数据的时候，发现无法加载响应数据。

这个数据就被称为 `React Server Components Payload`，简称 `RSC Payload`，其实你看这个地址的参数`?rsc=xxxx`其实就暗示了它跟 RSC 相关。查看返回的数据 ，你会发现这个数据很奇怪，既不是我们常见的 HTML、XML，也不是什么其他格式，这就是 React 定义的一种特殊的格式。

RSC Payload 包含哪些信息吗：

1.  服务端组件的渲染结果
2.  客户端组件的占位位置和引用文件
3.  从服务端组件传给客户端组件的数据

比如以 `0:` 开头的那行，根据其中的内容，可以判断出渲染的是笔记加载时的骨架图。以 `2:`开头的那行，渲染的则是笔记的具体内容。

**使用这种格式的优势在于它针对流做了优化，数据是分行的，它们可以以流的形式逐行从服务端发送给客户端，客户端可以逐行解析 RSC Payload，渐进式渲染页面。**

比如客户端收到 `0:`开头的这行，于是开始渲染骨架图。收到 `7:`开头的这行，发现需要下载 `static/chunks/app/note/[id]/page-5070a024863ac55b.js`，于是开始请求该 JS 文件，查看刚才的请求，也确实请求了该文件。收到 `2:`开头的这行，于是开始渲染笔记的具体内容。

因为我们特地设置了请求时间大于 5s，所以 `2:`开头的那行数据返回的时候肯定比 `0:`晚了 `5s`以上，这条请求的时长也确实大于了 5s，这也应证了 RSC Payload 服务端是逐行返回，客户端是逐行解析、渐进式渲染的。

我们接着讲 RSC Payload，那客户端获取到 RSC Payload 后还干了什么呢？其实就是根据 RSC Payload 重新渲染组件树，修改 DOM。但使用 RSC Payload 的好处在于组件树中的状态依然会被保持，比如左侧笔记列表的展开和收回就是一种客户端状态，当你新增笔记、删除笔记时，虽然组件树被重新渲染，但是客户端的状态依然会继续保持了。

这也被认为是 SSR 和 RSC 的最大区别，其实现的关键就在于服务端组件没有被渲染成 HTML，而是一种特殊的格式（RSC Payload）。这里让我们再复习下 SSR（传统的 SSR，想想 Pages Router 下的 SSR 实现） 和 RSC 的区别：

1.  RSC 的代码不会发送到客户端，但传统 SSR 所有组件的代码都会被发送到客户端
2.  RSC 可以在组件树中任意位置获取后端，传统 SSR 只能在顶层（getServerSideProps）访问后端
3.  服务器组件可以重新获取，而不会丢失其树内的客户端状态

注：这里虽然比较了 SSR 和 RSC，但并不是说明两者是冲突的，其实 SSR 和 RSC 是互补关系，是可以一起使用的，Next.js 中两者就是一起使用的。

## 路由缓存

点击切换不同的笔记，你会发现同样一条笔记，有时会触发数据的重新请求（出现了骨架图），但有的时候又没有，但有的时候又会重新出现（又出现了骨架图），这是为什么吗？

这就是 Next.js 提供的客户端路由缓存功能，客户端会缓存 RSC Payload 数据，所以当点击笔记后很快再次点击，这时就会从缓存中获取数据，那么问题来了，缓存的失效逻辑还记得吗？具体会缓存多久呢？我们在[缓存篇](https://juejin.cn/book/7307859898316881957/section/7309077169735958565#heading-20)中和大家讲过，回忆下基础知识：

路由缓存存放在浏览器的临时缓存中，有两个因素决定了路由缓存的持续时间：

- **Session，缓存在导航期间会持续存在，当页面刷新的时候会被清除**
- **自动失效期：单个路由段会在特定时长后自动失效，如果路由是静态渲染，持续 5 分钟，如果是动态渲染，持续 30s**

这个例子中因为我们用的是动态路由，是动态渲染，缓存持续 30s，所以首次点击笔记获取 RSC Payload 数据 30s 后再点击就会重新获取 RSC Payload。

小问题：以这个项目为例，如果点击笔记的时间算成 0s，因为请求时长大于 5s，假设 RSC Payload 在第 5s 完全返回，下次路由缓存失效重新获取的时间是大概在 30s 后还是 35s 后呢？

答案是 30s。以 RSC Payload 的返回时间为准，RSC Payload 是逐行返回的，所以点击的时候很快就有返回了。
