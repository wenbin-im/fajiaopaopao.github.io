title: iview 源码分析之 Form 组件
date: {{ 2020-04-07 }}
---

### 前言

在我们日常中后台的业务开发中，`<table>`作为展示数据项的一个常用组件，绝对占据了代码中的半壁江山，其中table，一般我们需要用到一下几个部分：

- table-head 显示表头
- table-body 显示表格内容
- 支持固定行列到左侧或者右侧
- 支持多选行

### created

```javascript
created () {
  if (!this.context) this.currentContext = this.$parent;
  this.showSlotHeader = this.$slots.header !== undefined;
  this.showSlotFooter = this.$slots.footer !== undefined;
  this.rebuildData = this.makeDataWithSortAndFilter();
}
```
- 保存对父组件的引用
- 控制slot Header & Footer 是否显示
- 初始化一份data数据

### mounted

```javascript
mounted () {
  this.handleResize();
  this.$nextTick(() => this.ready = true);

  on(window, 'resize', this.handleResize);
  this.observer = elementResizeDetectorMaker();
  this.observer.listenTo(this.$el, this.handleResize);

  this.$on('on-visible-change', (val) => {
      if (val) {
          this.$nextTick(()=>{
              this.handleResize();
          });
      }
  });
}
```

- 绑定resize相关的事件，进行动态的伸缩

### beforeDestroy

```javascript
beforeDestroy () {
  off(window, 'resize', this.handleResize);
  this.observer.removeListener(this.$el, this.handleResize);
}
```

- 卸载绑定的相关事件

#### Iview Table使用实例

```vue
<template>
  <Table :columns="columns1" :data="data1"></Table>
</template>

<script>
export default {
  data () {
    return {
      columns1: [
        {
          title: 'Name',
          key: 'name'
        },
        {
          title: 'Age',
          key: 'age'
        },
        {
          title: 'Address',
          key: 'address'
        }
      ],
      data1: [
        {
          name: 'John Brown',
          age: 18,
          address: 'New York No. 1 Lake Park',
          date: '2016-10-03'
        }
      ]
    }
  }
}
```

#### columns props 分析

```javascript
watch: {
  columns: {
  handler () {
      // todo 这里有性能问题，可能是左右固定计算属性影响的
      const colsWithId = this.makeColumnsId(this.columns);  
      this.allColumns = getAllColumns(colsWithId);
      this.cloneColumns = this.makeColumns(colsWithId);

      this.columnRows = this.makeColumnRows(false, colsWithId);
      this.leftFixedColumnRows = this.makeColumnRows('left', colsWithId);
      this.rightFixedColumnRows = this.makeColumnRows('right', colsWithId);
      this.rebuildData = this.makeDataWithSortAndFilter();
      this.handleResize();
    },
    deep: true
  }
}
```

- Table中对column属性设置了*watch deep*监听
- makeColumnsId 为column设置了一个随机6位的字符串标识：*__id*
- getAllColumns 对原有的column进行了一次拷贝
- makeColumns 对每个column设置相关的默认值
- makeColumnRows 设置左侧、右侧固定的列
- makeDataWithSortAndFilter 支持filters相关的配置
- handleResize 处理行列的宽度

#### data props 分析

```javascript
data: {
  handler () {
    const oldDataLen = this.rebuildData.length;
    this.objData = this.makeObjData();
    this.rebuildData = this.makeDataWithSortAndFilter();
    this.handleResize();
    if (!oldDataLen) {
        this.fixedHeader();
    }
    // here will trigger before clickCurrentRow, so use async
    setTimeout(() => {
      this.cloneData = deepCopy(this.data);
    }, 0);
  },
  deep: true
}
```

- makeObjData 支持数据的高亮、选择、展开/收起 效果
- makeDataWithSortAndFilter 根据columns 过滤 data数据
- cloneData 对传递的data进行深拷贝

上述有一个坑：vue table对传入的data进行了一次深拷贝，重新生成了数据`rebildData`，导致_rowKey每次递增，导致diff算法的优化失效，每次相关的数据需要重新render组件！！！其目的是为了引用数据之前修改，出现的的问题不容易追查。

#### table template分析

```html
<table-body
  ref="tbody"
  :draggable="draggable"
  :prefix-cls="prefixCls"
  :styleObject="tableStyle"
  :columns="cloneColumns"
  :data="rebuildData"
  :row-key="rowKey"
  :columns-width="columnsWidth"
  :obj-data="objData"></table-body>
```
```html

<template>
  <table cellspacing="0" cellpadding="0" border="0" :style="styleObject">
    <colgroup>
        <col v-for="(column, index) in columns" :width="setCellWidth(column)">
    </colgroup>
    <tbody :class="[prefixCls + '-tbody']">
      <template v-for="(row, index) in data">
        <table-tr
          :draggable="draggable"
          :row="row"
          :key="rowKey ? row._rowKey : index"
          :prefix-cls="prefixCls"
          @mouseenter.native.stop="handleMouseIn(row._index)"
          @mouseleave.native.stop="handleMouseOut(row._index)"
          @click.native="clickCurrentRow(row._index)"
          @dblclick.native.stop="dblclickCurrentRow(row._index)">
          <td v-for="column in columns" :class="alignCls(column, row)">
            <table-cell
              :fixed="fixed"
              :prefix-cls="prefixCls"
              :row="row"
              :key="column._columnKey"
              :column="column"
              :natural-index="index"
              :index="row._index"
              :checked="rowChecked(row._index)"
              :disabled="rowDisabled(row._index)"
              :expanded="rowExpanded(row._index)"
            ></table-cell>
          </td>
        </table-tr>
        <tr v-if="rowExpanded(row._index)" :class="{[prefixCls + '-expanded-hidden']: fixed}">
            <td :colspan="columns.length" :class="prefixCls + '-expanded-cell'">
                <Expand :key="rowKey ? row._rowKey : index" :row="row" :render="expandRender" :index="row._index"></Expand>
            </td>
        </tr>
      </template>
    </tbody>
  </table>
</template>
```

template中承载数据展示最重要的组件，传递了table中处理的数据。其中table-body 根据数据支持了对列相关的事件处理：`rowChecked`、`rowDisabled`、`rowExpanded`...

#### table table-tr

```html
<template>
    <tr :class="rowClasses(row._index)" :draggable="draggable" @dragstart="onDrag($event,row._index)" @drop="onDrop($event,row._index)" @dragover="allowDrop($event)" v-if="draggable"><slot></slot></tr>
    <tr :class="rowClasses(row._index)" v-else><slot></slot></tr>
</template>
<script>
    export default {
        props: {
            row: Object,
            prefixCls: String,
            draggable: Boolean
        },
        computed: {
            objData () {
                return this.$parent.objData;
            }
        },
        methods: {
            onDrag (e,index) {
                e.dataTransfer.setData('index',index);
            },
            onDrop (e,index) {
                const dragIndex = e.dataTransfer.getData('index');
                this.$parent.$parent.dragAndDrop(dragIndex,index);
                e.preventDefault();
            },
            allowDrop (e) {
                e.preventDefault();
            },
            rowClasses (_index) {
                return [
                    `${this.prefixCls}-row`,
                    this.rowClsName(_index),
                    {
                        [`${this.prefixCls}-row-highlight`]: this.objData[_index] && this.objData[_index]._isHighlight,
                        [`${this.prefixCls}-row-hover`]: this.objData[_index] && this.objData[_index]._isHover
                    }
                ];
            },
            rowClsName (_index) {
                return this.$parent.$parent.rowClassName(this.objData[_index], _index);
            },
        }
    };
</script>

```

和上面的table-body类似，都是容器组件，对传进来的数据做展示、操作相关的处理，比较简单明了。


#### table slot

```javascript
export default {
    name: 'TableSlot',
    functional: true,
    inject: ['tableRoot'],
    props: {
        row: Object,
        index: Number,
        column: {
            type: Object,
            default: null
        }
    },
    render: (h, ctx) => {
        return h('div', ctx.injections.tableRoot.$scopedSlots[ctx.props.column.slot]({
            row: ctx.props.row,
            column: ctx.props.column,
            index: ctx.props.index
        }));
    }
};
```

table支持Header、Footer slot之外，还支持对column中的slot支持，其实现的基础就是在源代码的`slot.js`中，通过inject和provide 将父组件Table中的数据传递了过来。

其具体的实现为一个函数式组件，通过inject的数据，进行了render达到了对表格更好的控制和展示。