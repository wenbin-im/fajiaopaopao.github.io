title: ZRender 源码分析 Part1 - 准备工作
date: 2021-01-02
categories:
- 可视化
- ZRender
tags:
- Canvas
- SVG
- ZRender
- ECharts
- JavaScript
language: zh-CN
toc: true
cover: /gallery/covers/zrender-part1.png
thumbnail: /gallery/covers/zrender-part1-thumbnail.png
---

ZRender是一个轻量级的Canvas类库，EChart就是在ZRender基础上建立的，给ECharts提供2D绘制能力。我平时的工作是负责BI系统的开发，需要用到图表库进行可视化展示，借此机会来更加深入了解平时用到的工具其底层原理。
<!-- more -->

## 简介

ZRender 是二维绘图引擎，它提供 Canvas、SVG、VML 多种渲染方式。ZRender 也是 ECharts 的渲染器。注意：文章的分析是基于ZRender 5.x版本进行。

- 文档官网：[https://ecomfe.github.io/zrender-doc/public/](https://ecomfe.github.io/zrender-doc/public/)
- GitHub主页：[https://github.com/ecomfe/zrender](https://github.com/ecomfe/zrender)
- Canvas API Doc：[https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API)
- SVG API Doc：[https://developer.mozilla.org/zh-CN/docs/Web/SVG](https://developer.mozilla.org/zh-CN/docs/Web/SVG)

##  Hello World

### 初始化 ZRender
在使用 ZRender 前需要初始化实例，具体方式是传入一个 DOM 容器：
```typescript
import zrender from "zrender";
const zr = zrender.init(document.getElementById('main'));
```
创建出的这个实例对应文档中 [zrender](https://ecomfe.github.io/zrender-doc/public/api.html#zrender-instance-api) 实例部分的方法和属性。

### 在场景中添加元素
ZRender 提供了将近 20 种图形类型，可以在文档 [zrender.Displayable](https://ecomfe.github.io/zrender-doc/public/api.html#zrenderdisplayable) 下找到。如果想创建其他图形，也可以通过 zrender.Path.extend 进行扩展。

以创建一个圆为例：
```javascript
const circle = new zrender.Circle({
  shape: {
    cx: 50,
    cy: 50,
    r: 40
  },
  style: {
    fill: "none",
    stroke: "#F00"
  }
});
zr.add(circle);

```
创建了一个圆心在 [50, 50] 位置，半径为 40 像素的圆，并将其添加到画布中。

### 修改图形元素属性
通过 `a = new zrender.XXX` 方法创建了图形元素之后，可以用 `a.shape` 等形式获取到创建时输入的属性，但是如果需要对其进行修改，应该使用 `a.attr(key, value)` 的形式修改，否则不会触发图形的重绘。例子：

```javascript
circle.attr("shape", {
  r: 50 // 只更新半径大小
});
```

### DEMO
<iframe src="https://codesandbox.io/embed/zrender-hello-world-do706?autoresize=1&fontsize=14&hidenavigation=1&theme=light&view=preview"
     style="width:100%; height:500px; border:0; border-radius: 4px; overflow:hidden;"
     title="zrender-hello-world"
     allow="accelerometer; ambient-light-sensor; camera; encrypted-media; geolocation; gyroscope; hid; microphone; midi; payment; usb; vr; xr-spatial-tracking"
     sandbox="allow-forms allow-modals allow-popups allow-presentation allow-same-origin allow-scripts"
   ></iframe>



## 目录介绍
``` shell
src
├─ animation  # 动画操作相关
├─ contain    # 是否包含判断utils
├─ core       # 核心业务逻辑
├─ dom        # dom事件处理
├─ graphic    # 各种图形的实体，图形实体，分而治之的图形策略，可定义扩展
├─ mixin      # 需要进行混入的utils
├─ canvas     # Canvas操作封装
├─ svg        # SVG操作封装
├─ vml        # VML操作封装，VML是早期IE浏览器绘制矢量图形的语言，后被SVG替代
├─ tool       # 绘画拓展工具
├─ Element.ts # ZRender中最基础的元素
├─ Handler.ts # Controller层，控制层，事件交互处理，实现dom元素的模拟封装
├─ Painter.ts # View层，视图层，元素生命周期管理，视图渲染，绘制和更新控制
├─ Storage.ts # Model层，shape数据管理层
├─ Layer.ts   # 图层管理
├─ config.ts  # 配置信息
├─ export.ts  # 导出
└─ zrender.ts # 入口
```

## 设计思想
总的来说，MVC核心封装实现图形存储、视图渲染和交互控制：

- Stroage(M) : shape数据CURD管理
- Painter(V) : canvase元素生命周期管理，视图渲染，绘画，更新控制
- Handler(C) : 事件交互处理，实现完整dom事件模拟封装
- shape : 图形实体，分而治之的图形策略，可定义扩展
- tool : 绘画扩展相关实用方法，工具及脚手架

{% img "box px-0 py-0 ml-auto mr-auto" /assets/2021-01-02/zrender.png 360 '"ZRender" "ZRender"' %}

## 源码分析

### 入口分析
我们可以从`package.json`的`main`字段或者`build/build.js`构建文件了解到，ZRender的入口文件在`/index.ts`中：

{% codeblock index.ts lang:typescript %}
export * from './src/zrender';
export * from './src/export';

import './src/canvas/canvas';
import './src/svg/svg';
// import './src/vml/vml'; // 5.x注释了对IE VML的支持
{% endcodeblock %}

上述的两个export语句，是用来初始化ZRender，那么ZRender到底是什么，它是如何工作的，我们来一探究竟吧。


### ZRender的入口 
通过上面的分析，具体`ZRender`的入口是：`src/zrender.ts`，我们先具体看下这的实现逻辑：

{% codeblock src/zrender.ts lang:typescript %}
let instances: { [key: number]: ZRender } = {};

function delInstance(id: number) {
    delete instances[id];
}

class ZRender { ... }

/**
 * Initializing a zrender instance
*/
export function init(dom: HTMLElement, opts?: ZRenderInitOpt) {
    const zr = new ZRender(zrUtil.guid(), dom, opts);
    instances[zr.id] = zr;
    return zr;
}

/**
 * Dispose zrender instance
 */
export function dispose(zr: ZRender) {
    zr.dispose();
}

/**
 * Dispose all zrender instances
 */
export function disposeAll() {
    for (let key in instances) {
        if (instances.hasOwnProperty(key)) {
            instances[key].dispose();
        }
    }
    instances = {};
}

/**
 * Get zrender instance by id
 */
export function getInstance(id: number): ZRender {
    return instances[id];
}

export function registerPainter(name: string, Ctor: PainterBaseCtor) {
    painterCtors[name] = Ctor;
}

export const version = '5.0.1';
{% endcodeblock %}


这里的关键部分有: 
- `class ZRender {...}`: ZRender类，后续再展开分析
- `init()`: 初始化ZRender实例，并以*key-value*的形式存储到*instances*对象中
- `dispose()`: 销毁 ZRender 实例
- `disposeAll()`: 销毁*instances*中所有的ZRender实例
- `getInstance()、delInstance()`: 获取or删除*instances*中的实例
- `registerPainter()`: 注册Canvas、SVG视图层逻辑

### ZRender的export
除了上面的`src/zrender.ts`模块，入口文件还导出了`src/exports`，具体的代码如下：

{% codeblock src/exports.ts lang:typescript %}
/**
 * Do not mount those modules on 'src/zrender' for better tree shaking.
 */

import * as zrUtil from './core/util';
import * as matrix from './core/matrix';
import * as vector from './core/vector';
import * as colorTool from './tool/color';
import * as pathTool from './tool/path';
import {parseSVG} from './tool/parseSVG';
import {morphPath} from './tool/morphPath';

export {default as Point} from './core/Point';

export {default as Element} from './Element';

export {default as Group} from './graphic/Group';
export {default as Path} from './graphic/Path';
export {default as Image} from './graphic/Image';
export {default as CompoundPath} from './graphic/CompoundPath';
export {default as TSpan} from './graphic/TSpan';
export {default as IncrementalDisplayable} from './graphic/IncrementalDisplayable';
export {default as Text} from './graphic/Text';

export {default as Arc} from './graphic/shape/Arc';
export {default as BezierCurve} from './graphic/shape/BezierCurve';
export {default as Circle} from './graphic/shape/Circle';
export {default as Droplet} from './graphic/shape/Droplet';
export {default as Ellipse} from './graphic/shape/Ellipse';
export {default as Heart} from './graphic/shape/Heart';
export {default as Isogon} from './graphic/shape/Isogon';
export {default as Line} from './graphic/shape/Line';
export {default as Polygon} from './graphic/shape/Polygon';
export {default as Polyline} from './graphic/shape/Polyline';
export {default as Rect} from './graphic/shape/Rect';
export {default as Ring} from './graphic/shape/Ring';
export {default as Rose} from './graphic/shape/Rose';
export {default as Sector} from './graphic/shape/Sector';
export {default as Star} from './graphic/shape/Star';
export {default as Trochoid} from './graphic/shape/Trochoid';

export {default as LinearGradient} from './graphic/LinearGradient';
export {default as RadialGradient} from './graphic/RadialGradient';
export {default as Pattern} from './graphic/Pattern';
export {default as BoundingRect} from './core/BoundingRect';
export {default as OrientedBoundingRect} from './core/OrientedBoundingRect';

export {matrix};
export {vector};
export {colorTool as color};
export {pathTool as path};
export {zrUtil as util};

export {parseSVG};
export {morphPath};

export {default as showDebugDirtyRect} from './debug/showDebugDirtyRect';
{% endcodeblock %}

可以看到，`export.ts`的职责是用来导出模块到外部。其中包括`core`、`tool`中的工具函数方法，以及ZRender抽象出来的各种`shape`图形模块，方便以`zrender.xx`方式进行调用。

另外，模块的第一行注释写到：*Do not mount those modules on 'src/zrender' for better tree shaking*，相比在`src/zrender.ts`中一股脑进行导出和调用，`export.ts`导出的都是无副作用的函数，更加有利于构建工具对代码进行静态分析进行Tree Shaking，避免ECharts使用ZRender时，打包了未实际使用的代码。

### ZRender的画笔 🖌

前期基本工作已经介绍完了，ZRender是如何将图形绘制出来的呢？
其实现在前端主流的2D绘制方式有两种：Canvas和SVG，那么ZRender就需要分别封装Canvas和SVG各自的API，来抹平不同绘制方式在使用时候的差异：

**导入画笔** 
{% codeblock index.ts lang:typescript %}
export * from './src/zrender';
...
import './src/canvas/canvas';
import './src/svg/svg';
// import './src/vml/vml';

{% endcodeblock %}

**注册Canvas画笔** 
{% codeblock src/canvas/canvas.ts lang:typescript %}
import './graphic';
import {registerPainter} from '../zrender';
import Painter from './Painter';

registerPainter('canvas', Painter);
{% endcodeblock %}

**注册SVG画笔** 
{% codeblock src/svg/svg.ts lang:typescript %}
import './graphic';
import {registerPainter} from '../zrender';
import Painter from './Painter';

registerPainter('svg', Painter);
{% endcodeblock %}

上面👆代码的主要作用是：在初始化执行完zrender后，调用`registerPainter()`方法来把画笔🖌注册到全局的zrender中，其中：
- **graphic模块**: 封装形状、图片、文本...绘制函数
- **Painter模块**: 提供给zrender进行图形绘制

**VML**
可以看到5.x版本注释了VML(Vector Markup Language)画笔，不了解VML的可以将VML理解为SVG的祖先，用于兼容低版本IE浏览器。[VML维基百科](https://zh.wikipedia.org/wiki/VML%E8%AF%AD%E8%A8%80)定义：
> Vector Markup Language（VML）是一种XML语言用于绘制矢量图形（vector graphics）。1998年VML建议书由微软、Macromedia等向W3C提出审核。VML遭到拒绝，因为Adobe、Sun等提出了PGML[1]计划书。这两套标准后来合并成更具潜力的SVG。

## 总结

至此，ZRender的初始化过程基本介绍完毕，本节只对ZRender的主线进行了粗浅的介绍，后续将一层一层的探索ZRender的秘密，揭开它的神秘面纱。

## 参考链接

- [InfoQ: 可视化了解一下？ ECharts 4.0最全技术攻略](https://mp.weixin.qq.com/s?__biz=MjM5MDE0Mjc4MA==&mid=2651006663&idx=4&sn=1833bf61617c12aa5f4880508b25a9f9&chksm=bdbede948ac95782ff28cba0c35dd973a8a3b4094b4bd5113cfcf3b6838382e865dea2a9a092&mpshare=1&scene=1&srcid=0408sz6o6RYNUMdtftmDyJgb#rd)
- [FunDeug: 配置Tree Shaking来减少JavaScript的打包体积](https://blog.fundebug.com/2018/08/15/reduce-js-payload-with-tree-shaking/)