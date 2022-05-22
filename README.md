# 如何在React中实现PDF文件预览

## 写在前面

`PDF`预览是我们平时工作中比较常见的需求，所以我写了这篇文章，帮助你快速的在 react 项目中实现预览 PDF 的功能

下面是本次教程中使用的技术栈

- Vite
- React
- Typescript
- pdf.js


## 快速搭建项目

```bash
> yarn create vite pdf-preview --template react-ts
```

现在我们安装下 [pdf.js](https://github.com/mozilla/pdf.js)

通过官网的介绍，并没有发现 npm 的下载方式，这时候很多人估计就会直接安装 umd 版本的了，其实使用一个库除了看文档，看官方案例也是非常重要的，通过源代码下的 `examples/webpack/main.js` 文件，我们看到 `pdfjs-dist` 这个npm包，我们来下载

```bash
> yarn add pdfjs-dist
```

然后按照自己的习惯组织下文件目录

```bash
.
├── components
│   └── PDFRender
│       └── index.tsx
├── main.tsx
├── App.tsx
└── vite-env.d.ts
```

## 开发预览组件

### 渲染第一页
这里我新建了一个 PDFRender 组件，先来实现一个最简单的，将 PDF 的第一页渲染出来

```ts
import * as pdf from 'pdfjs-dist'
import pdfWorker from 'pdfjs-dist/build/pdf.worker.js?url'
import React, { useLayoutEffect, useRef } from "react";

pdf.GlobalWorkerOptions.workerSrc = pdfWorker;

export const PDFRender: React.FC<{ src: string }> = (props) => {
  const canvasRef = useRef<HTMLCanvasElement | null>(null)
  useLayoutEffect(() => {
    pdf
    .getDocument(props.src)
    .promise
    .then(pdfDocument => {
      return pdfDocument.getPage(1);
    })
    .then((pdfPage) => {
      const viewport = pdfPage.getViewport({ scale: 1.0 });
      const canvas = canvasRef.current;
      if (!canvas) {
        return Promise.reject()
      }
      canvas.width = viewport.width
      canvas.height = viewport.height;
      const ctx = canvas.getContext("2d") as CanvasRenderingContext2D
      const renderTask = pdfPage.render({
        canvasContext: ctx,
        viewport,
      });
      return renderTask.promise;
    })
    .catch(err => {
      console.log(err)
    })
  }, [])
  return (
    <canvas ref={canvasRef}/>
  )
}
```

细心的同学可能发现了这两行代码

```ts
import pdfWorker from 'pdfjs-dist/build/pdf.worker.js?url'
pdf.GlobalWorkerOptions.workerSrc = pdfWorker;
```

这是因为pdf的交互容易堵塞JS，所以 pdf.js 使用了 `web worker` 技术优化了性能。

最后我们使用下这个组件，看下效果

```ts
import { PDFRender } from "./components/PDFRender";

const pdfFilePath = '/kalacloud-demo.pdf'

export const App = () => {
  return (
    <PDFRender src={pdfFilePath} />
  )
}
```

效果如下

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h2h2jalg9qj21hc0ojq4w.jpg)

### 渲染整个PDF并翻页

想渲染全部页面其实很简单，按照上面的思路，获取到页数，直接循环渲染就好了

```ts
import * as pdf from 'pdfjs-dist'
import pdfWorker from 'pdfjs-dist/build/pdf.worker.js?url'
import { useEffect, useRef, useState } from "react";

pdf.GlobalWorkerOptions.workerSrc = pdfWorker;

export const usePDFData = (options: { src: string, scale?: number }) => {
  const previewUrls = useRef<string[]>([])
  const urls = useRef<string[]>([])
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    urls.current = []
    setLoading(true)
    ;(async () => {
      // 这里千万别解构，会导致 this 指向错误
      const pdfDocument = await pdf.getDocument(options.src).promise
      const task = new Array(pdfDocument.numPages).fill(null)
      await Promise.all(task.map(async (_, i) => {
        const page = await pdfDocument.getPage(i + 1)
        const viewport = page.getViewport({ scale: options.scale || 2 })
        const canvas = document.createElement('canvas')

        canvas.width = viewport.width
        canvas.height = viewport.height
        const ctx = canvas.getContext("2d") as CanvasRenderingContext2D
        const renderTask = page.render({
          canvasContext: ctx,
          viewport,
        });
        await renderTask.promise;
        // 分别获取不同尺寸的图片，一个用来预览一个用来展示
        urls.current[i] = canvas.toDataURL('image/jpeg', 1)
        previewUrls.current[i] = canvas.toDataURL('image/jpeg', 0.5)
      }))
      setLoading(false)
    })()
  }, [options.src])

  return {
    loading,
    urls: urls.current,
    previewUrls: previewUrls.current,
  }
}
```

接下来我们实现滚动翻页功能

1. 点击对应页滚动到指定的位置
2. 滚动到对应位置，高亮当前页

先看下最终的效果

![](https://tva1.sinaimg.cn/large/e6c9d24ely1h2h90wmumxg20rs0crhdu.gif)

首先实现点击滚动到对应的位置，非常的简单，利用 scrollIntoView api 可以快速定位到指定位置

```ts
  const goPage = (i: number) => {
    setCurrentPage(i)
    document.querySelectorAll('.page')[i]!.scrollIntoView({ behavior: 'smooth' })
  }
```

再来实现下滚动位置自动高亮页数

本质上是使用 `IntersectionObserver` api 来完成，监听每个页面的可见性，当可见性大于 0.5 也就是有一半的内容展示在视口里面则就确定为当前页

```ts
  const io = useRef(new IntersectionObserver((entries) => {
    entries.forEach(item => {
      item.intersectionRatio >= 0.5 && setCurrentPage(Number(item.target.getAttribute('index')))
    })
  }, {
    threshold: [0.5]
  }))
```

## 源代码

