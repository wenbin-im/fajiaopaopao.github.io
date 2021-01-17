title: ZRender 源码分析 Part2 - ZRender Class
date: 2021-01-15
categories:
- 可视化
- ZRender
tags:
- ZRender
- ECharts
- JavaScript
language: zh-CN
toc: true
cover: /gallery/covers/zrender-part2.png
thumbnail: /gallery/covers/zrender-part2-thumbnail.png
---

从[上篇文章](https://www.ifajiao.com/zrender-source-code-part1/)了解了`ZRender`初始化的工作流程，这篇将继续学习`ZRender`这个基础类的源码设计。
<!-- more -->

## 简介
我们之前提到的[初始化 ZRender](https://www.ifajiao.com/zrender-source-code-part1/#%E5%88%9D%E5%A7%8B%E5%8C%96-ZRender)中，初始化一个ZRender实例并没有到`new`语句，其实是在`init()`中实例化`class ZRender`，然后导出外部调用。

{% codeblock src/zrender.ts lang:typescript %}
export function init(dom: HTMLElement, opts?: ZRenderInitOpt) {
    const zr = new ZRender(zrUtil.guid(), dom, opts);
    instances[zr.id] = zr;
    return zr;
}
{% endcodeblock %}


## ZRender的定义

{% codeblock src/zrender.ts lang:typescript %}
class ZRender {
    id: number // unique id
    dom: HTMLElement // zrender容器dom

    storage: Storage // mvc model
    painter: PainterBase // mvc view
    handler: Handler // mvc control
    animation: Animation // 动画 TODO:学习

    private _sleepAfterStill = 10;

    private _stillFrameAccum = 0;

    private _needsRefresh = true
    private _needsRefreshHover = true

    /**
     * @description If theme is dark mode. It will determine the color strategy for labels.
     * @extra: echarts5 新能力，自行判断标签文本颜色
     */
    private _darkMode = false;

    private _backgroundColor: string | GradientObject | PatternObject; -->

    constructor(id: number, dom: HTMLElement, opts?: ZRenderInitOpt) {...}

    // ============= 图形存储相关 =============

    /**
     * @extra: 添加元素 -> 添加到storage -> this.refresh()
     */
    add(el: Element) {...}

    /**
     * @extra: 删除元素 -> 从storage删除 -> this.refresh()
     */
    remove(el: Element) {...}

    // ============= TODO: 待学习 =============

    /**
     * Change configuration of layer
    */
    configLayer(zLevel: number, config: LayerConfig) {...}

    // ============= 背景颜色相关 =============

    /**
     * @description 设置画布背景颜色
     * @extra: backgroundColor -> set Cavas/Svg backgroundColor -> 存储到this._backgroundColor -> refresh()
     */
    setBackgroundColor(backgroundColor: string | GradientObject | PatternObject) {...}


    /**
      * @description 获取画布背景颜色
      * @extra: return this._backgroundColor
      */
    getBackgroundColor() {...}


    // ============= 暗黑模式相关 =============
    /**
     * Force to set dark mode
     */
    setDarkMode(darkMode: boolean) {...}

    isDarkMode() {...}


    // ============= 刷新绘制相关 =============

    /**
     * Repaint the canvas immediately
     */
    refreshImmediately(fromInside?: boolean) {...}

    /**
     * Mark and repaint the canvas in the next frame of browser
     */
    refresh() {...}

    /**
     * Perform all refresh
     */
    flush() {...}

    private _flush(fromInside?: boolean) {...}

    /**
     * Set sleep after still for frames.
     * Disable auto sleep when it's 0.
     */
    setSleepAfterStill(stillFramesCount: number) {...}

    /**
     * Wake up animation loop. But not render.
     */
    wakeUp() {...}

    /**
     * Refresh hover in next frame
     */
    refreshHover() {...}

    /**
     * Refresh hover immediately
     */
    refreshHoverImmediately() {...}

    /**
     * Stop and clear all animation immediately
     */
    clearAnimation() {...}



    // ============= DOM容器相关 =============

    /**
     * Resize the canvas.
     * Should be invoked when container size is changed
     * @description handler和painter处理resize
     */
    resize(opts?: { width?: number| string height?: number | string }) {...}

    /**
     * Get container width
     */
    getWidth(): number {...}

    /**
     * Get container height
     */
    getHeight(): number {...}



    // ============= 其他 =============
    /**
     * Converting a path to image.
     * It has much better performance of drawing image rather than drawing a vector path.
     */
    pathToImage(e: Path, dpr: number) {...}

    /**
     * Set default cursor
     * @param cursorStyle='default' 例如 crosshair
     */
    setCursorStyle(cursorStyle: string) {...}

    /**
     * Find hovered element
     * @param x
     * @param y
     * @return {target, topTarget}
     */
    findHover(x: number, y: number): { target: Displayable, topTarget: Displayable } {...}



    // ============= 自定义事件相关 =============

    on<Ctx>(eventName: eventHandler: , context?: Ctx): this {...}

    /**
     * Unbind event
     * @param eventName Event name
     * @param eventHandler Handler function
     */
    off(eventName?: string, eventHandler?: EventCallback<unknown, unknown>) {...}

    /**
     * Trigger event manually
     *
     * @param eventName Event name
     * @param event Event object
     */
    trigger(eventName: string, event?: unknown) {...}


    // ============= 卸载、清理相关 =============

    /**
     * Clear all objects and the canvas.
     */
    clear() {...}

    /**
     * Dispose self.
     */
    dispose() {...}
}
{% endcodeblock %}

从上面对`class ZRender`的分析了解到，zrender实例包含了以下三大功能：
- 数据存储层：
  - 管理存储画布内的元素，对应Storage相关的处理
- 视图绘制层：
  - Painter绘制图形元素
  - 绘图容器dom的属性感知和DOM元素创建、修改、resize
  - 进行refresh、resize、clear、dispose、flush等绘制操作
- 逻辑控制层：
  - click、mouse、hover等逻辑控制

