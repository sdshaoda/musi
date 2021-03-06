<template>
  <scroll class="listview" ref="listview" :data="data" :listenScroll="true" :probeType="3" @scroll="scroll">
    <ul>
      <li v-for="group in data" :key="group.title" class="list-group" ref="listGroup">
        <h2 class="list-group-title">{{group.title}}</h2>
        <ul>
          <li v-for="item in group.items" :key="item.id" class="list-group-item" @click="selectItem(item)">
            <img class="avatar" v-lazy="item.smallAvatar">
            <span class="name">{{item.name}}</span>
          </li>
        </ul>
      </li>
    </ul>
    <!-- 右侧的快速索引 -->
    <div class="list-shortcut" @touchstart.stop.prevent="onShortcutTouchStart" @touchmove.stop.prevent="onShortcutTouchMove">
      <ul>
        <li v-for="(item, index) in shortcutList" :key="item" :data-index="index" class="item" :class="{'current': currentIndex === index}">
          {{item}}
        </li>
      </ul>
    </div>
    <!-- 固定的索引 title -->
    <div class="list-fixed" v-show="fixedTitle" ref="fixedTitle">
      <h1 class="fixed-title">{{fixedTitle}}</h1>
    </div>
    <div class="loading-container" v-show="!data.length">
      <loading></loading>
    </div>
  </scroll>
</template>

<script>
import Scroll from 'base/scroll/scroll'
import Loading from 'base/loading/loading'
// import { getData } from 'common/js/dom'

const ANCHOR_HEIGHT = 18
const TITLE_HEIGHT = 30

export default {
  props: {
    data: {
      type: Array,
      default: []
    }
  },
  data() {
    return {
      scrollY: -1,
      currentIndex: 0,
      diff: -1
    }
  },
  computed: {
    shortcutList() {
      return this.data.map((item) => {
        return item.title.substr(0, 1)
      })
    },
    fixedTitle() {
      if (this.scrollY > 0) {
        return ''
      }
      return this.data[this.currentIndex] ? this.data[this.currentIndex].title : ''
    }
  },
  created() {
    // 不希望被 Vue 响应化，所以不放在 data 函数中
    this.touch = {}             // 触摸相关
    this.listHeight = []        // 索引高度数组
  },
  methods: {
    selectItem(item) {
      // 发送 'select' 事件，参数为 item
      this.$emit('select', item)
    },
    onShortcutTouchStart(e) {
      // 从DOM属性中获得的是字符串，转换为数字
      let anchorIndex = parseInt(e.target.dataset.index)
      // （多点）触控的第一个点的对象信息
      let firstTouch = e.touches[0]

      this.touch = {}
      this.touch.y1 = firstTouch.pageY
      this.touch.anchorIndex = anchorIndex

      this._scrollTo(anchorIndex)
    },
    onShortcutTouchMove(e) {
      let firstTouch = e.touches[0]
      this.touch.y2 = firstTouch.pageY

      // Math.floor()  | 0
      let delta = (this.touch.y2 - this.touch.y1) / ANCHOR_HEIGHT | 0
      let anchorIndex = this.touch.anchorIndex + delta

      this._scrollTo(anchorIndex)
    },
    scroll(pos) {
      this.scrollY = pos.y
    },
    // 刷新 scroll
    refresh() {
      this.$refs.listview.refresh()
    },
    _scrollTo(index) {
      if (!index && index !== 0) {
        return
      }
      if (index < 0) {
        index = 0
      } else if (index > this.listHeight.length - 2) {
        index = this.listHeight.length - 2
      }

      this.scrollY = -this.listHeight[index]
      // 滚动到相应位置，第二个参数为0表示无缓动动画效果
      this.$refs.listview.scrollToElement(this.$refs.listGroup[index], 0)
    },
    _calculateListHeight() {
      this.listHeight = [0]

      const list = this.$refs.listGroup
      for (let i = 0, height = 0; i < list.length; i++) {
        height += list[i].clientHeight
        this.listHeight.push(height)
      }
    }
  },
  watch: {
    data() {
      // 确保 data 已正确插入
      setTimeout(() => {
        this._calculateListHeight()
      }, 20)
    },
    // 参数 newScrollY 为 scrollY 变化后的新值
    scrollY(newScrollY) {
      const listHeight = this.listHeight
      // 顶部以上，newScrollY > 0
      if (newScrollY > 0) {
        this.currentIndex = 0
        return
      }
      // 中间
      for (let i = 0; i < listHeight.length - 1; i++) {
        let height1 = listHeight[i]
        let height2 = listHeight[i + 1]
        if (-newScrollY >= height1 && -newScrollY < height2) {
          this.currentIndex = i
          this.diff = height2 + newScrollY
          return
        }
      }
      // 底部且超出
      this.currentIndex = listHeight - 2
    },
    diff(newDiff) {
      let fixedTop = (newDiff > 0 && newDiff < TITLE_HEIGHT) ? newDiff - TITLE_HEIGHT : 0
      if (this.fixedTop === fixedTop) {
        return
      }
      this.fixedTop = fixedTop
      this.$refs.fixedTitle.style.transform = `translate3d(0, ${fixedTop}px, 0)`
    }
  },
  components: {
    Scroll,
    Loading
  }
}
</script>

<style scoped lang="stylus">
  @import "~common/stylus/variable"

  .listview
    position: relative
    width: 100%
    height: 100%
    overflow: hidden
    background: $color-background
    .list-group
      padding-bottom: 30px
      .list-group-title
        height: 30px
        line-height: 30px
        padding-left: 20px
        font-size: $font-size-small
        color: $color-text-l
        background: $color-highlight-background
      .list-group-item
        display: flex
        align-items: center
        padding: 20px 0 0 30px
        .avatar
          width: 50px
          height: 50px
          border-radius: 50%
        .name
          margin-left: 20px
          color: $color-text-l
          font-size: $font-size-medium
    .list-shortcut
      position: absolute
      z-index: 30
      right: 0
      top: 50%
      transform: translateY(-50%)
      width: 20px
      padding: 20px 0
      border-radius: 10px
      text-align: center
      background: $color-background-d
      font-family: Helvetica
      .item
        padding: 3px
        line-height: 1
        color: $color-text-l
        font-size: $font-size-small
        &.current
          color: $color-theme
    .list-fixed
      position: absolute
      top: 0
      left: 0
      width: 100%
      .fixed-title
        height: 30px
        line-height: 30px
        padding-left: 20px
        font-size: $font-size-small
        color: $color-text-l
        background: $color-highlight-background
    .loading-container
      position: absolute
      width: 100%
      top: 50%
      transform: translateY(-50%)
</style>
