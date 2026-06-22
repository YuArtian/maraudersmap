#React #性能优化

> <https://react.dev/learn/react-compiler/introduction>
> <https://zh-hans.react.dev/learn/react-compiler/introduction>

## 相关文章

- [[ReactCompiler实现原理]] - 真实编译产物（`_c(n)` 缓存槽），不是直接插入 useMemo
- [[ReactCompiler与useEffect依赖]] - 为什么 effect 依赖项仍然要手写 useMemo

# Memoize

记忆化 [[Marauder'sMap/Wordbook/Memoize|Memoize]] 是 React 中重要的概念

对于函数组件相关的 api 有：

- useMemo 缓存值
- useCallback 缓存函数
- memo 缓存组件

之前的用法大概如下：

```jsx

import { useMemo, useCallback, memo } from 'react'; 

const ExpensiveComponent = memo(function ExpensiveComponent({ data, onClick }) {

const processedData = useMemo(() => {  

 return expensiveProcessing(data);

}, [data]);

const handleClick = useCallback((item) => {  

 onClick(item.id);  

}, [onClick]);  

return (
 <div>
  {
   processedData.map(item => (
    <Item key={item.id} onClick={() => handleClick(item)} />
   ))
  }
 </div>
);
});

```

# React Compiler

现在有了 React Compiler 则不需要手动写入这些 api
只需要安装 `babel-plugin-react-compiler@latest`
配合 `eslint` 插件 `eslint-plugin-react-hooks@latest` 能更好的检查语法

> <mark style="background: #FFB8EBA6;">注意 目前 compiler 还没有 swc 的版本，请注意使用 babel</mark>
> 比如： vite 中应该使用 @vitejs/plugin-react ， 而不是 @vitejs/plugin-react-swc
