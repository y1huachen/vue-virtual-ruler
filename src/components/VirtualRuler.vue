<template>
  <div class="virtual-ruler" :data-vr-id="id">
    <div ref="scroll" class="virtual-ruler-scroll" :style="style" @scroll="scroll">
      <div class="virtual-ruler-body" :style="{ minWidth: getPx(size) }">
        <div class="virtual-ruler-body-item" ref="placeholder"/>
        <div
          v-for="(item, index) of pool"
          :key="index"
          :style="{ transform: `translateX(${(item.index) * gap + halfWidth}px)`, width: `${gap}px` }"
          class="virtual-ruler-body-item"
        >
          <div class="ruler-wrapper" :style="{ height: itemDIVHeight }">
            <div :style="getItemStyle(item.index)" />
          </div>
          <div v-if="labelShow(item.index)" :style="labelStyle" class="virtual-ruler-label">
            <slot name="label" :value="item.gapValue" :index="item.index">
              {{ item.gapValue }}
            </slot>
          </div>
        </div>
      </div>
    </div>
    <div class="virtual-ruler-point" :class="{ 'virtual-ruler-point-even': isEven }" :style="_pointStyle" />
  </div>
</template>

<script>
let id = 0;
function getNextId () {
  id += 1;
  return String(id);
}
export default {
  name: 'VirtualRuler',
  props: {
    // 默认值
    defaultValue: Number,
    // 最小刻度值
    min: {
      type: Number,
      default: 0
    },
    // 最大刻度值
    max: {
      type: Number,
      default: 100
    },
    visible: {
      type: Number,
      default: 10
    },
    // 刻度线宽度
    itemWidth: {
      type: [Number, String],
      default: -1
    },
    // 刻度线颜色
    itemColor: {
      type: String,
      default: '#cccccc'
    },
    // 刻度线高度函数默认为 8/15/20
    itemHeight: {
      type: Function,
      default: (index) => {
        if (index % 10 === 0) {
          return 20;
        }
        if (index % 5 === 0) {
          return 15;
        }
        return 8;
      }
    },
    // 刻度线最高高度 默认为 itemHeight 的最高高度
    itemMaxHeight: {
      type: [Number, String],
      default () {
        const all = Array.from({ length: this.max + 1 - this.min }).map((_, index) => this.itemHeight(index));
        return Math.max(...all);
      }
    },
    // 刻度标线宽度
    pointWidth: {
      type: [Number, String],
      default: 1
    },
    // 刻度标线颜色
    pointColor: {
      type: String,
      default: '#00C5CD'
    },
    // 刻度标线样式
    pointStyle: Object,
    // 刻度值label间隔
    labelGap: {
      type: Number,
      default: 10
    },
    // 是否显示刻度值label
    showLabel: {
      type: Boolean,
      default: true
    },
    // 刻度值的倍数，例如刻度值为 100，间隔为0.1则值为100 * 0.1,默认为1
    valueGap: {
      type: Number,
      default: 1
    },
    // 缓冲数量，预先渲染
    buffer: {
      type: Number,
      default: 10
    },
    // 刻度尺高度
    height: {
      type: [Number, String],
      required: true
    },
    // 刻度尺上边框
    topBorder: {
      type: Boolean,
      default: false
    },
    // 刻度尺上边框宽度
    topBorderWidth: {
      type: [Number, String],
      default: 1
    },
    // 刻度尺上边框颜色
    topBorderColor: {
      type: String,
      default: '#cccccc'
    },
    // 刻度尺背景渐变
    linearGradient: Boolean,
    // 刻度尺背景渐变方向
    linearGradientDirectionVar: {
      type: Array,
      default: () => [
        'to', 'right'
      ]
    },
    // 刻度尺背景渐变颜色
    linearGradientColorVar: {
      type: Array,
      default: () => [
        'rgba(255, 255, 255, 1)',
        'rgba(255, 255, 255, 0) 25%',
        'rgba(255, 255, 255, 0) 85%',
        'rgba(255, 255, 255, 1) 100%'
      ]
    },
    // 是否自动监听元素大小改变事件
    autoResize: Boolean,
    // 刻度值label样式
    labelStyle: Object
  },
  data () {
    return {
      startIndex: 0,
      scrollTimer: null,
      val: this.defaultValue || this.min,
      isTouch: false,
      fireScrollEnd: null,
      width: document.body.clientWidth,
      observer: null,
      id: getNextId(),
      styleSheet: this.getStyleSheet(),
      elementWidth: this.itemWidth,
      isFromInput: false
    };
  },
  computed: {
    isEven() {
      return this.viewCount % 2 === 0
    },
    gap() {
      const elementWidth = this.elementWidth
      const viewCount = this.viewCount
      const contentWidth = this.width
      const interval = (contentWidth - viewCount * elementWidth) / (viewCount - 1)
      return interval + elementWidth
    },
    /**
     * 一半可视宽度
     * @returns {number}
     */
    halfWidth () {
      const width = (this.viewCount - 1) / 2 * this.gap
      return this.isEven ? width + this.elementWidth / 2 : width;
    },
    /**
     * 所有刻度尺列表
     * @returns {{gapValue: number, index: number, value: number}[]}
     */
    list () {
      return Array.from({ length: this.max + 1 - this.min }).map((_, index) => {
        const value = index + this.min;
        return {
          value,
          index,
          gapValue: this.getRealValue(value)
        };
      });
    },
    decimalCount () {
      return this.getDecimalCount(this.valueGap);
    },
    /**
     * 可视刻度尺数量
     * @returns {number}
     */
    viewCount () {
      return this.visible;
    },
    /**
     * 可视范围开始刻度尺值的下标
     * @returns {number}
     */
    start () {
      return Math.max(0, Math.floor(this.startIndex - this.viewCount / 2 - this.buffer));
    },
    /**
     * 可视范围结束刻度尺值的下标
     * @returns {number}
     */
    end () {
      return Math.min(this.start + this.viewCount + this.buffer * 2, this.list.length);
    },
    /**
     * 可视范围的刻度尺列表
     * @returns {{gapValue: number, index: number, value: number}[]}
     */
    pool () {
      return this.list.slice(this.start, this.end);
    },
    /**
     * 总刻度尺的元素大小
     * @returns {number}
     */
    size () {
      return this.list.length * this.gap + this.width - this.gap;
    },
    /**
     * 刻度尺最大高度
     * @returns {string}
     */
    itemDIVHeight () {
      return this.getPx(this.itemMaxHeight);
    },
    /**
     * 每个刻度尺宽度
     * @returns {string}
     * @private
     */
    _itemWidth () {
      const { gap } = this;
      return this.getPx(gap);
    },
    /**
     * 刻度尺标线样式
     * @returns {{borderLeft: string, left: string, height: string}}
     * @private
     */
    _pointStyle () {
      const width = this.getPx(this.pointWidth);
      return {
        width,
        height: this.itemDIVHeight,
        backgroundColor: this.pointColor,
        ...(this.pointStyle || {})
      };
    },
    /**
     * 刻度尺容器样式
     * @returns {{borderTop: string, height: string}}
     */
    style () {
      return {
        height: this.getPx(this.height),
        borderTop: this.topBorder ? `${this.getPx(this.topBorderWidth)} solid ${this.topBorderColor}` : 'none'
      };
    },
  },
  watch: {
    /**
     * 实时发送当前值变化
     * @param value
     */
    val (value) {
      this.$emit('change', this.getRealValue(value));
    },
    /**
     * 设置背景渐变
     */
    linearGradient () {
      this.setAfterStyle();
    },
    /**
     * 设置背景渐变方向
     */
    linearGradientDirectionVar () {
      this.setAfterStyle();
    },
    /**
     * 设置背景渐变颜色
     */
    linearGradientColorVar () {
      this.setAfterStyle();
    },
    autoSize () {
      this.setAutoResize();
    }
  },
  mounted () {
    if (this.elementWidth === -1) {
      const placeholder = this.$refs.placeholder
      this.elementWidth = placeholder.clientWidth
    }
    this.setAutoResize();
    this.resize();
    setTimeout(() => this.setValue(this.val));
    this.setAfterStyle();
    this.$el.addEventListener('touchstart', this.touchStart.bind(this));
    this.$el.addEventListener('touchend', this.touchEnd.bind(this));
  },
  methods: {
    getStyleSheet () {
      for (let i = 0; i < document.styleSheets.length; i++) {
        const sheet = document.styleSheets.item(i);
        if (!sheet.href) {
          return sheet;
        }
      }
      const style = document.createElement('style');
      document.head.appendChild(style);
      return this.getStyleSheet();
    },
    async setAutoResize () {
      if (this.autoResize) {
        const elementResizeDetectorMaker = await import('element-resize-detector');
        this.observer = elementResizeDetectorMaker.default();
        this.observer.uninstall(this.$el);
        this.observer.listenTo(this.$el, this.resize.bind(this));
      } else {
        if (this.observer) {
          this.observer.uninstall(this.$el);
        }
      }
    },
    resize () {
      this.width = this.$refs.scroll.clientWidth;
    },
    touchStart () {
      this.isTouch = true;
      this.isFromInput = false
    },
    touchEnd () {
      this.isTouch = false;
      if (this.fireScrollEnd) {
        const event = this.fireScrollEnd;
        this.fireScrollEnd = null;
        this.scrollEnd(event);
      }
    },
    getRealValue (value) {
      return Number((value * this.valueGap).toFixed(this.decimalCount));
    },
    /**
     * 获取像素样式
     * @param value
     * @param unit
     * @returns {*}
     */
    getPx (value, unit = 'px') {
      return typeof value === 'number' ? `${value}${unit}` : value;
    },
    /**
     * 获取刻度线样式
     * @param index
     * @returns {{}|{borderLeft: string, height: *}}
     */
    getItemStyle (index) {
      if (index < 0) {
        return {};
      }
      const { itemColor, itemHeight } = this;
      const height = this.getPx(itemHeight(index));
      return {
        width: `${4}px`,
        height,
        backgroundColor: itemColor,
      };
    },
    /**
     * 刻度值label是否显示
     * @param index
     * @returns {boolean|boolean}
     */
    labelShow (index) {
      return this.showLabel && (index % this.labelGap === 0);
    },
    setAfterStyle () {
      let style;
      if (!this.linearGradient) {
        style = 'content: none';
      } else {
        const direction = this.linearGradientDirectionVar.join(' ');
        const color = this.linearGradientColorVar.join(',');
        style = `background: linear-gradient(${direction}, ${color}); content: ""`;
      }
      const selector = `.virtual-ruler[data-vr-id="${this.id}"]::after`;
      this.deleteAfterCSSRule(selector);
      this.styleSheet.addRule(selector, style);
    },
    deleteAfterCSSRule (selector) {
      for (let i = 0; i < this.styleSheet.rules.length; i++) {
        if (this.styleSheet.rules.item(i).selectorText === selector) {
          this.styleSheet.deleteRule(i);
          return;
        }
      }
    },
    getDecimalCount (num) {
      const str = String(num);
      const index = str.indexOf('.');
      return Math.max(str.length - 1 - index, 0);
    },
    /**
     * 设置刻度值并滑动刻度尺标线到该位置
     * @param value
     */
    setValue (value) {
      const { valueGap } = this;
      const offset = value / valueGap - this.min
      console.log(offset, value)
      this.$refs.scroll.scrollLeft = offset * this.gap;
      this.isFromInput = true
      this.val = offset + this.min
    },
    scrollEnd () {
      this.setValue(this.getRealValue(this.val));
    },
    scroll (event) {
      console.log(event)
      const { scrollLeft } = event.target;
      if (scrollLeft >= this.list.length * this.gap) {
        return;
      }
      clearTimeout(this.scrollTimer);
      if (!this.isFromInput) {
        this.val = Math.round(scrollLeft / this.gap) + this.min;
      }
      this.startIndex = Math.floor(scrollLeft / this.gap);
      this.scrollTimer = setTimeout(() => {
        if (this.isTouch) {
          this.fireScrollEnd = event;
        } else {
          this.scrollEnd(event);
        }
      }, 200);
    }
  }
};
</script>

<style>
.virtual-ruler {
  position: relative;
  width: 100%;
}
.virtual-ruler::after {
  content: "";
  position: absolute;
  z-index: 1;
  top: 0;
  right: 0;
  pointer-events: none;
  width: 100%;
  height: 100%;
}

.virtual-ruler-scroll {
  overflow-x: auto;
  position: relative;
}
.virtual-ruler-scroll::-webkit-scrollbar {
  display: none;
}

.virtual-ruler-point {
  position: absolute;
  left: 50%;
  top: 0;
  transform: translateX(-50%);
}
.virtual-ruler-point-even {
  transform: none;
}

.virtual-ruler-body {
  position: relative;
  display: flex;
  height: 100%;
}
.virtual-ruler-body-item {
  position: absolute;
  top: 0;
  left: 0;
  will-change: transform;
  background-color: red;
  width: 4px;
}

.virtual-ruler-label {
  position: absolute;
  text-align: center;
  width: 50px;
  left: -25px;
  color: #cccccc;
  font-size: 15px;
}
</style>