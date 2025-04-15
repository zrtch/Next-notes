## 多用服务端组件

- 我们引入 `day.js` 的 SidebarNoteList 组件使用的是服务端渲染，这意味着 `day.js` 的代码并不会被打包到客户端的 bundle 中。

- 但如果你现在在 SidebarNoteList 组件的顶部添加 `'use client'`，声明为客户端组件，你会发现立刻就多了 day.js

这就是使用 React Server Compoent 的好处之一，服务端组件的代码不会打包到客户端的 bundle 中：
