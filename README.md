# gif-cursor

将 gif 指针转为纯 css / js 实现，而无需引入npm包

## 试用地址



## 使用方法

### html

```html
<html>
    <head>
        <!-- css放哪都行 -->
        <link rel="stylesheet" href="./cursors/cursor.css">
    </head>
    <body>
        <!-- 如果有js，则放在body的开头 -->
        <script src="./cursors/cursor.js"></script>
    </body>
</html>
```

### vue或其他

```js
// 我不怎么写前端，关于动态指针你去npm上找找应该有更好的轮子
import './cursors/cursor.css'
import './cursors/cursor.js'
```

## 注意事项

- gif 作为指针应尽量使用 32x32 大小，否则容易渲染错误
- 动画时长从 gif 信息中获取，默认情况下不需要更改

### 单一指针

- `*{}`通用选择器可能会与少数控件样式冲突，使用后需要检查你的页面是否有样式错误，并添加`:not()`排除冲突类

### 多类型指针

- 如果你想设置多种类型的指针，有两种方案可以选择

    1. 不使用 js，仅引入 css，然后将对应指针的类添加到对应的元素上（维护效率低，但稳定可控）
    2. 同时引入 js 和 css，在每次页面加载时遍历元素并添加对应指针类（维护效率高，但可能影响页面性能且不可控）

## 本地运行

### Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```

##

项目灵感来自于给[蛍川](https://space.bilibili.com/330525028)写的[个人主页](https://hotarukawa.live/#/)中实现的动态指针

关注蛍川喵，关注蛍川谢谢喵