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
