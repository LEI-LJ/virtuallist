<template>
  <div ref="listContainerRef" class="virtual-list-container" :style="{height}" @scroll="onScroll">
    <div ref="listPlaceRef" class="virtual-list-place"></div>
    <div ref="listRef" class="virtual-list">
      <div
          ref="itemsRef"
          class="virtual-list-item"
          v-for="(item, itemIndex) in visibleList"
          :key="item._key"
          :id="item._key"
      >
        <slot name="default" :item="item" :index="itemIndex"></slot>
      </div>
    </div>
    <slot name="footer"></slot>
  </div>
</template>
<script>
export default {
  props: {
    // 首尾缓存比例 = (首或者尾高度 + 滚动视窗高度) / 滚动视窗高度
    // 自动高度开启时需要调大缓存区比例（视窗内能容纳的越多该值应该越大）
    bufferScale: {
      type: Number,
      default: 0.4,
      requred: false
    },
    // 数据
    dataSource: {
      type: Array,
      requred: true
    },
    // 滚动视窗高度
    height: {
      type: String,
      default: '100%',
      requred: false
    },
    // （非必填）列表每一项高度（不填时需开启itemAutoHeight）
    itemHeight: {
      type: Number,
      default: 60,
      requred: false
    },
    // 是否自动计算每一项高度
    itemAutoHeight: {
      type: Boolean,
      default: false
    },
    // 到达底部回调
    onReachBottom: Function
  },
  data() {
    return {
      screenHeight: 0,
      startIndex: 0,
      endIndex: 0,
      positions: []
    }
  },
  computed: {
    // 处理后的列表
    _listData() {
      return this.dataSource.map((item, index) => ({
        ...item,
        _key: `${index}`
      }))
    },
    // 拿到
    anchorPoint() {
      return this.positions.length ? this.positions[this.startIndex] : null
    },
    visibleCount() {
      return Math.ceil(this.screenHeight / this.itemHeight)
    },
    aboveCount() {
      return Math.min(this.startIndex, Math.ceil(this.bufferScale * this.visibleCount))
    },
    belowCount() {
      return Math.min(this._listData.length - this.endIndex, Math.ceil(this.bufferScale * this.visibleCount))
    },
    visibleList() {
      const startIndex = this.startIndex - this.aboveCount
      const endIndex = this.endIndex + this.belowCount
      return this._listData.slice(startIndex, endIndex)
    }
  },
  mounted() {
    this.init()
  },
  updated() {
    console.log('updated')
    // 列表数据长度不等于缓存长度
    if (this._listData.length !== this.positions.length) {
      this.initPositions()
    }
    this.$nextTick(function () {
      if (!this.$refs.itemsRef || !this.$refs.itemsRef.length) {
        return
      }
      // 获取真实元素大小，修改对应的尺寸缓存
      if (this.itemAutoHeight) {
        this.updateItemsSize()
      }

      // 更新列表总高度
      console.log(this.positions, ' this.positions')
      const height = this.positions[this.positions.length - 1].bottom
      this.$refs.listPlaceRef.style.height = height + 'px'
      // 更新真实偏移量
      this.setStartOffset()
    })
  },
  methods: {
    init() {
      this.initPositions()
      this.screenHeight = this.$refs.listContainerRef.clientHeight
      this.startIndex = 0
      this.endIndex = this.visibleCount
      this.setStartOffset()
    },
    initPositions() {
      this.positions = this._listData.map((_, index) => ({
            index,
            height: this.itemHeight,
            top: index * this.itemHeight,
            bottom: (index + 1) * this.itemHeight
          })
      )
    },
    updateItemsSize() {
      const nodes = this.$refs.itemsRef
      nodes.forEach((node) => {
        const rect = node.getBoundingClientRect()
        const height = rect.height
        const index = +node.id.split('_')[0]
        const oldHeight = this.positions[index].height
        const dValue = oldHeight - height
        if (dValue) {
          this.positions[index].bottom = this.positions[index].bottom - dValue
          this.positions[index].height = height
          this.positions[index].over = true

          for (let k = index + 1; k < this.positions.length; k++) {
            this.positions[k].top = this.positions[k - 1].bottom
            this.positions[k].bottom = this.positions[k].bottom - dValue
          }
        }
      })
    },
    // 设置开始的偏移量
    setStartOffset() {
      let startOffset
      if (this.startIndex >= 1) {
        const size = this.positions[this.startIndex].top - (this.positions[this.startIndex - this.aboveCount] ? this.positions[this.startIndex - this.aboveCount].top : 0)
        startOffset = this.positions[this.startIndex - 1].bottom - size
      } else {
        startOffset = 0
      }
      this.startOffset = startOffset
      this.$refs.listRef.style.transform = `translate3d(0,${startOffset}px,0)`
    },
    getStartIndex(scrollTop = 0) {
      return this.binarySearch(this.positions, scrollTop)
    },
    binarySearch(list, value) {
      let start = 0
      let end = list.length - 1
      let tempIndex = null
      while (start <= end) {
        const midIndex = parseInt((start + end) / 2)
        const midValue = list[midIndex].bottom
        if (midValue === value) {
          return midIndex + 1
        } else if (midValue < value) {
          start = midIndex + 1
        } else if (midValue > value) {
          if (tempIndex === null || tempIndex > midIndex) {
            tempIndex = midIndex
          }
          end = end - 1
        }
      }
      return tempIndex
    },
    // 执行滚动事件
    onScroll() {
      const scrollTop = this.$refs.listContainerRef.scrollTop
      const scrollHeight = this.$refs.listContainerRef.scrollHeight
      // 如果 当前滚动的
      if (scrollTop > this.anchorPoint.bottom || scrollTop < this.anchorPoint.top) {
        this.startIndex = this.getStartIndex(scrollTop)
        this.endIndex = this.startIndex + this.visibleCount
        this.setStartOffset()
      }
      if (scrollTop + this.screenHeight > scrollHeight - 50) {
        this.onReachBottom && this.onReachBottom()
      }
    }
  }
}
</script>
<style>
.virtual-list-container {
  overflow: auto;
  position: relative;
}

.virtual-list-place {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  z-index: -1;
}

.virtual-list {
  position: absolute;
  left: 0;
  top: 0;
  right: 0;
  z-index: 1;
}
</style>
