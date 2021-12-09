<template>
  <div 
    class="visual-list" 
    ref="viewport" 
    @scroll="scrollChange"
  >
    <div class="scroll-bar" ref="scrollBar" />
    <div 
      class="scroll-list" 
      ref="scrollList"
      :style="{transform: `translate3d(0, ${offset}px, 0)`}"
    >
      <div 
        v-for="item in visibleItems" 
        :key="item.id"
        :style="itemStyle"
        :vid="item.id"
        ref="items"
      >
        <slot v-bind="item"></slot>
      </div>
    </div>
  </div>
</template>

<script>
import { throttle } from './utils'
export default {
  name: 'VisualList',
  props: {
    size: {
      type: Number,
      default: 40
    },
    // 最少加载的个数
    remain: {
      type: Number,
      default: 10
    },
    items: {
      required: true,
      type: Array,
      default: () => []
    },
    // 不固定高度
    variable: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      start: 0,
      end: this.remain,
      offset: 0,
      positions: []
    }
  },
  computed: {
    // 预渲染的期望是，在可视区域的前后都各渲染最多remain个
    prevCount() {
      // start 3，remain 8，此时就取三 这就是取min的原因
      return Math.min(this.start, this.remain);
    },
    nextCount() {
      // items 150 end 148 remain 8，这时只能取 150 - 148 = 2
      return Math.min(this.remain, this.items.length - this.end);
    },
    // 最终加载的items
    visibleItems() {
      let start = this.start - this.prevCount;
      let end = this.end + this.nextCount;
      return this.items.slice(start, end)
    },
    itemStyle() {
      let base = {
        width: '100%'
      }
      if (!this.variable) {
        base.height = this.size + 'px'
      }
      return base
    }
  },
  mounted() {
    // 设置可视区域的高度
    this.$refs.viewport.style.height = this.size * this.remain + 'px'
    // 设置滚动条scrollbar的高度
    this.$refs.scrollBar.style.height = this.size * this.items.length + 'px'
    // 先给每一项缓存一个假的高度
    this.cacheList()
  },
  updated() {
    // 防止dom没加载好
    this.$nextTick(() => {
      const nodes = this.$refs.items
      if (!nodes?.length) {
        return
      }
      nodes.forEach(node => {
        // 拿到元素的高度等信息 
        let { height } = node.getBoundingClientRect()
        let vid = node.getAttribute('vid')
        // 拿到再positions中的索引
        let posiIndex = this.positions.findIndex(item => item.id == vid)
        // 拿到对应的那一项
        let currPosi = this.positions[posiIndex]
        // 拿到之前的高度
        let oldHeight = currPosi.height
        // 求出之前高度和现在高度的差值
        let diff = oldHeight - height
        // 如果新老高度没变 那么当前项和当前项之后的所有项都不用修改
        if (diff != 0) {
          // 修改高度，因为再初始化的时候都有这些属性 所以不用splice也可以
          currPosi.height = height
          // 高度变了对应的bottom肯定也响应的增加或减少了 用老的bottom - 老新高度的差 就是bottom增加或减少的部分
          // 如果老的短新的长 那么diff为负值，负负得正所以bottom还是增加了
          currPosi.bottom = currPosi.bottom - diff
          // 高度增加top值不会变，因为是往底部撑开，所以要修改当前索引后面的所有项的top和bottom值来保持同步
          // 相当于链表操作 为什么只考虑后面部分的更新  因为首次滚动肯定是从前往后滚动
          for (let i = posiIndex + 1; i < this.positions.length; i++) {
            let posiItem = this.positions[i]
            // 下一项的top就是上一项的bottom
            posiItem.top = this.positions[i - 1].bottom
            // 当前项的高度改变了多少 那么当前项和之后的所有项的bottom都是改变新老高度的差值
            posiItem.bottom = posiItem.bottom - diff
          }
        }
      })
      // 重新计算scrollbar的高度 不难相想出 最后一项的bottom就是scrollbar的高度
      // 因为bottom缓存的其实就是所有item的高度之和
      this.$refs.scrollBar.style.height = this.positions[this.positions.length - 1].bottom + 'px'
    })
  },
  methods: {
    // 先基于size缓存一个假的 当触发滚动时，动态的去修改
    cacheList() {
      this.positions = this.items.map((item, index) => ({
        id: item.id,
        height: this.size,
        top: index * this.size,
        bottom: (index + 1) * this.size
      }))
    },
    getStartIndex(value) {
      let start = 0
      // 因为我们计算的是索引 所以需要减去1
      let end = this.items.length - 1
      // 保存范围
      let temp = null
      // 使用二分查找 找到当前value对应的索引
      while (start <= end) {
        let middleIndex = parseInt((start + end) / 2)
        let middleValue = this.positions[middleIndex].bottom
        if (middleValue == value) {
          return middleIndex + 1
        } else if (middleValue < value) { // 开始向右查找
          start = middleIndex + 1
        } else if (middleValue > value) { // 开始向左查找
          // 保证最终返回的索引永远是 大于value的下一项的索引
          // 因为这个索引是可视区域中的start
          if (temp == null || temp > middleIndex) {
            temp = middleIndex
          }
          end = middleIndex - 1
        }
      }
      return temp
    },
    // 添加节流 单位时间内只执行一次
    scrollChange: throttle(function () {
      console.log("scroll")
      // 拿到viewport顶部卷去的高度
      let scrollTop = this.$refs.viewport.scrollTop
      // 说明是不指定高度的 需要我们自己计算滚动了多少个了
      if (this.variable) {
        this.start = this.getStartIndex(scrollTop)
        this.end = this.start + this.remain
        // 设置偏移量， 如果可以找到就返回top值 找不到说明索引越界了（这里只会小于0），滚动到顶部了， 返回0即可
        this.offset = this.positions[this.start - this.prevCount] ? this.positions[this.start - this.prevCount].top : 0
      } else {
        let start = scrollTop / this.size
        // 算出应该要从第几个开始显示了
        this.start = Math.floor(start)
        // 如果被卷去的高度可以被整除 说明可视区域刚好可以完全将remain个显示出来，
        // 如果不然因为顶部有部分是被卷去的 所以底部会有空白，因此在底部补一个保证可视区域被填充满，因为做了预渲染所以这里就没必要加了
        // this.end = start == this.start ? this.start + this.remain : this.start + this.remain + 1
        this.end = this.start + this.remain
        // 调整可视区域距离顶部的位置 因为做了预渲染 所以还要减去预渲染头部的高度
        this.offset = this.start * this.size - this.size * this.prevCount
      }
    }, 200)
  }
}
</script>

<style scoped lang="scss">
.visual-list {
  position: relative;
  width: 100%;
  overflow-y: scroll;
  .scroll-list {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
  }
}
</style>
