<template>
  <div class="vue-waterfall" :style="style">
    <slot></slot>
  </div>
</template>

<style>
.vue-waterfall {
  position: relative;
  /*overflow: hidden; cause clientWidth = 0 in IE if height not bigger than 0 */
}
</style>

<script>
const MOVE_CLASS_PROP = "_wfMoveClass";

export default {
  props: {
    // 窗口大小改变时自动reflow
    autoResize: {
      default: true,
    },
    // 两次reflow之间的最小时间间隔
    interval: {
      default: 200,
      validator: (val) => val >= 0,
    },
    // 对齐方式
    align: {
      default: "left",
      validator: (val) => ~["left", "right", "center"].indexOf(val),
    },
    // 垂直方向或水平方向，默认垂直
    line: {
      default: "v",
      validator: (val) => ~["v", "h"].indexOf(val),
    },
    // 标准宽度(垂直)/高度(水平)
    lineGap: {
      required: true,
      validator: (val) => val >= 0,
    },
    // 最小宽度/高度
    minLineGap: {
      validator: (val) => val >= 0,
    },
    // 最大宽度/高度
    maxLineGap: {
      validator: (val) => val >= 0,
    },
    // 水平方向上单个slot的最大宽度
    singleMaxWidth: {
      validator: (val) => val >= 0,
    },
    // line = 'v' 时是否调整slot的高度
    fixedHeight: {
      default: false,
    },
    // line = 'v' 时slot在水平方向上的放大比例
    grow: {
      validator: (val) => val instanceof Array,
    },
    // 监听的内容
    watch: {
      default: () => ({}),
    },
  },
  data: () => ({
    style: {
      height: "",
      overflow: "",
    },
    token: null,
  }),
  methods: {
    reflowHandler,
    autoResizeHandler,
    reflow,
  },
  created() {
    this.virtualRects = [];
    this.$on("reflow", () => {
      this.reflowHandler();
    });
    this.$watch(
      () => (
        this.align,
        this.line,
        this.lineGap,
        this.minLineGap,
        this.maxLineGap,
        this.singleMaxWidth,
        this.fixedHeight,
        this.watch
      ),
      this.reflowHandler
    );
    this.$watch("grow", this.reflowHandler);
  },
  mounted() {
    this.$watch("autoResize", this.autoResizeHandler);
    on(this.$el, getTransitionEndEvent(), tidyUpAnimations, true);
    this.autoResizeHandler(this.autoResize);
  },
  beforeDestroy() {
    this.autoResizeHandler(false);
    off(this.$el, getTransitionEndEvent(), tidyUpAnimations, true);
  },
};

function autoResizeHandler(autoResize) {
  if (autoResize === false || !this.autoResize) {
    off(window, "resize", this.reflowHandler, false);
  } else {
    on(window, "resize", this.reflowHandler, false);
  }
}

function tidyUpAnimations(event) {
  let node = event.target;
  let moveClass = node[MOVE_CLASS_PROP];
  if (moveClass) {
    removeClass(node, moveClass);
  }
}

function reflowHandler() {
  clearTimeout(this.token);
  this.token = setTimeout(this.reflow, this.interval);
}

function reflow() {
  if (!this.$el) {
    return;
  }
  let width = this.$el.clientWidth;
  // 获取每个waterfall-slot的meta
  let metas = this.$children.map((slot) => slot.getMeta());
  metas.sort((a, b) => a.order - b.order);
  // virtualRects和metas一一对应
  this.virtualRects = metas.map(() => ({}));
  calculate(this, metas, this.virtualRects);
  setTimeout(() => {
    if (isScrollBarVisibilityChange(this.$el, width)) {
      calculate(this, metas, this.virtualRects);
    }
    this.style.overflow = "hidden";
    render(this.virtualRects, metas);
    this.$emit("reflowed", this);
  }, 0);
}

function isScrollBarVisibilityChange(el, lastClientWidth) {
  return lastClientWidth !== el.clientWidth;
}

function calculate(vm, metas, styles) {
  let options = getOptions(vm);
  let processor =
    vm.line === "h" ? horizontalLineProcessor : verticalLineProcessor;
  processor.calculate(vm, options, metas, styles);
}

// 处理waterfall的props
function getOptions(vm) {
  const maxLineGap = vm.maxLineGap ? +vm.maxLineGap : vm.lineGap;
  return {
    align: ~["left", "right", "center"].indexOf(vm.align) ? vm.align : "left",
    line: ~["v", "h"].indexOf(vm.line) ? vm.line : "v",
    lineGap: +vm.lineGap,
    minLineGap: vm.minLineGap ? +vm.minLineGap : vm.lineGap,
    maxLineGap: maxLineGap,
    singleMaxWidth: Math.max(vm.singleMaxWidth || 0, maxLineGap),
    fixedHeight: !!vm.fixedHeight,
    grow: vm.grow && vm.grow.map((val) => +val),
  };
}

// 垂直瀑布流
var verticalLineProcessor = (() => {
  function calculate(vm, options, metas, rects) {
    let width = vm.$el.clientWidth;
    let grow = options.grow;
    // strategy = {
    //   width: [], // 每块的宽度数组
    //   count: xxx, // 总块数
    //   left: xxx // 整体的left值
    // }
    let strategy = grow
      ? getRowStrategyWithGrow(width, grow) // 根据grow计算
      : getRowStrategy(width, options); // 根据lineGap计算
    // 当前每列的tops
    let tops = getArrayFillWith(0, strategy.count);
    metas.forEach((meta, index) => {
      // 找出tops中最小值的索引
      let offset = tops.reduce(
        (last, top, i) => (top < tops[last] ? i : last),
        0
      );
      // 当前块对应的width
      let width = strategy.width[offset % strategy.count];
      // 计算当前块对应的top left width height
      let rect = rects[index];
      rect.top = tops[offset];
      rect.left =
        strategy.left + (offset ? sum(strategy.width.slice(0, offset)) : 0);
      rect.width = width;
      // 调整height
      rect.height =
        meta.height * (options.fixedHeight ? 1 : width / meta.width);
      // 更新tops
      tops[offset] = tops[offset] + rect.height;
    });
    // waterfall的高度为tops中的最大值
    vm.style.height = Math.max.apply(Math, tops) + "px";
  }

  function getRowStrategy(width, options) {
    let count = width / options.lineGap;
    let slotWidth;
    // 根据不同情况，自适应count和slotWidth
    if (options.singleMaxWidth >= width) {
      count = 1;
      slotWidth = Math.max(width, options.minLineGap);
    } else {
      let maxContentWidth = options.maxLineGap * ~~count;
      let minGreedyContentWidth = options.minLineGap * ~~(count + 1);
      let canFit = maxContentWidth >= width;
      let canFitGreedy = minGreedyContentWidth <= width;
      if (canFit && canFitGreedy) {
        count = Math.round(count);
        slotWidth = width / count;
      } else if (canFit) {
        count = ~~count;
        slotWidth = width / count;
      } else if (canFitGreedy) {
        count = ~~(count + 1);
        slotWidth = width / count;
      } else {
        count = ~~count;
        slotWidth = options.maxLineGap;
      }
      if (count === 1) {
        slotWidth = Math.min(width, options.singleMaxWidth);
        slotWidth = Math.max(slotWidth, options.minLineGap);
      }
    }
    return {
      width: getArrayFillWith(slotWidth, count), // 等宽
      count: count,
      left: getLeft(width, slotWidth * count, options.align), // 根据对齐方式获取left值
    };
  }

  function getRowStrategyWithGrow(width, grow) {
    let total = sum(grow);
    return {
      width: grow.map((val) => (width * val) / total),
      count: grow.length,
      left: 0,
    };
  }

  return {
    calculate,
  };
})();

// 水平瀑布流
var horizontalLineProcessor = (() => {
  function calculate(vm, options, metas, rects) {
    let width = vm.$el.clientWidth;
    // waterfall-slot的总数
    let total = metas.length;
    let top = 0;
    let offset = 0;
    while (offset < total) {
      // strategy = {
      //   left: xxx, // 整行的left值
      //   count: xxx, // 整行的总块数
      //   height: xxx // 整行的height值
      // }
      let strategy = getRowStrategy(width, options, metas, offset);
      for (let i = 0, left = 0, meta, rect; i < strategy.count; i++) {
        meta = metas[offset + i];
        rect = rects[offset + i];
        rect.top = top;
        rect.left = strategy.left + left;
        // width根据height缩放
        rect.width = (meta.width * strategy.height) / meta.height;
        rect.height = strategy.height;
        left += rect.width;
      }
      offset += strategy.count;
      top += strategy.height;
    }
    vm.style.height = top + "px";
  }

  // 根据内容的width获得最初的count
  // 对比count和count-1两种情况，满足最大最小值要求且最接近标准值lineGap和行宽rowWidth(宽度必须小于rowWidth)的作为最终的实际量
  function getRowStrategy(width, options, metas, offset) {
    let greedyCount = getGreedyCount(width, options.lineGap, metas, offset);
    let lazyCount = Math.max(greedyCount - 1, 1);
    let greedySize = getContentSize(width, options, metas, offset, greedyCount);
    let lazySize = getContentSize(width, options, metas, offset, lazyCount);
    let finalSize = chooseFinalSize(lazySize, greedySize, width);
    let height = finalSize.height;
    let fitContentWidth = finalSize.width;
    if (finalSize.count === 1) {
      fitContentWidth = Math.min(options.singleMaxWidth, width);
      height = (metas[offset].height * fitContentWidth) / metas[offset].width;
    }
    return {
      left: getLeft(width, fitContentWidth, options.align),
      count: finalSize.count,
      height: height,
    };
  }

  // 根据行高lineGap和内容高度比例缩放内容的宽度，计算一行最多能放的count
  function getGreedyCount(rowWidth, rowHeight, metas, offset) {
    let count = 0;
    for (
      let i = offset, width = 0;
      i < metas.length && width <= rowWidth;
      i++
    ) {
      width += (metas[i].width * rowHeight) / metas[i].height;
      count++;
    }
    return count;
  }

  // 获取每行的属性
  // {
  //   cost: xxx,
  //   count: xxx,
  //   width: xxx,
  //   height: xxx
  // }
  function getContentSize(rowWidth, options, metas, offset, count) {
    // 根据标准行高lineGap缩放后的所有内容的宽度
    let originWidth = 0;
    for (let i = count - 1; i >= 0; i--) {
      let meta = metas[offset + i];
      originWidth += (meta.width * options.lineGap) / meta.height;
    }
    // 将所有内容的宽度缩放至行宽，行高等比缩放，得到缩放后的实际高度
    let fitHeight = (options.lineGap * rowWidth) / originWidth;
    let canFit =
      fitHeight <= options.maxLineGap && fitHeight >= options.minLineGap;
    if (canFit) {
      return {
        cost: Math.abs(options.lineGap - fitHeight),
        count: count,
        width: rowWidth,
        height: fitHeight,
      };
    } else {
      let height =
        originWidth > rowWidth ? options.minLineGap : options.maxLineGap;
      return {
        cost: Infinity,
        count: count,
        width: (originWidth * height) / options.lineGap,
        height: height,
      };
    }
  }

  function chooseFinalSize(lazySize, greedySize, rowWidth) {
    if (lazySize.cost === Infinity && greedySize.cost === Infinity) {
      // 根据最大最小值调整的情况，取最接近行宽rowWidth的
      return greedySize.width < rowWidth ? greedySize : lazySize;
    } else {
      // 根据标准高度的情况，取最接近标准高度lineGap的
      return greedySize.cost >= lazySize.cost ? lazySize : greedySize;
    }
  }

  return {
    calculate,
  };
})();

function getLeft(width, contentWidth, align) {
  switch (align) {
    case "right":
      return width - contentWidth;
    case "center":
      return (width - contentWidth) / 2;
    default:
      return 0;
  }
}

function sum(arr) {
  return arr.reduce((sum, val) => sum + val);
}

function render(rects, metas) {
  let metasNeedToMoveByTransform = metas.filter((meta) => meta.moveClass);
  // 更新之前的rects
  let firstRects = getRects(metasNeedToMoveByTransform);
  applyRects(rects, metas);
  // 更新之后的rects
  let lastRects = getRects(metasNeedToMoveByTransform);
  metasNeedToMoveByTransform.forEach((meta, i) => {
    meta.node[MOVE_CLASS_PROP] = meta.moveClass;
    // transform移动
    setTransform(meta.node, firstRects[i], lastRects[i]);
  });
  document.body.clientWidth; // forced reflow
  // 添加类名 移除transform
  metasNeedToMoveByTransform.forEach((meta) => {
    addClass(meta.node, meta.moveClass);
    clearTransform(meta.node);
  });
}

function getRects(metas) {
  return metas.map((meta) => meta.vm.rect);
}

// 更新vm的rect和真实dom的top left width height
function applyRects(rects, metas) {
  rects.forEach((rect, i) => {
    let style = metas[i].node.style;
    metas[i].vm.rect = rect;
    for (let prop in rect) {
      style[prop] = rect[prop] + "px";
    }
  });
}

function setTransform(node, firstRect, lastRect) {
  let dx = firstRect.left - lastRect.left;
  let dy = firstRect.top - lastRect.top;
  let sw = firstRect.width / lastRect.width;
  let sh = firstRect.height / lastRect.height;
  node.style.transform = node.style.WebkitTransform = `translate(${dx}px,${dy}px) scale(${sw},${sh})`;
  node.style.transitionDuration = "0s";
}

function clearTransform(node) {
  node.style.transform = node.style.WebkitTransform = "";
  node.style.transitionDuration = "";
}

function getTransitionEndEvent() {
  let isWebkitTrans =
    window.ontransitionend === undefined &&
    window.onwebkittransitionend !== undefined;
  let transitionEndEvent = isWebkitTrans
    ? "webkitTransitionEnd"
    : "transitionend";
  return transitionEndEvent;
}

/**
 * util
 */

function getArrayFillWith(item, count) {
  let getter = typeof item === "function" ? () => item() : () => item;
  let arr = [];
  for (let i = 0; i < count; i++) {
    arr[i] = getter();
  }
  return arr;
}

function addClass(elem, name) {
  if (!hasClass(elem, name)) {
    let cur = attr(elem, "class").trim();
    let res = (cur + " " + name).trim();
    attr(elem, "class", res);
  }
}

function removeClass(elem, name) {
  let reg = new RegExp("\\s*\\b" + name + "\\b\\s*", "g");
  let res = attr(elem, "class").replace(reg, " ").trim();
  attr(elem, "class", res);
}

function hasClass(elem, name) {
  return new RegExp("\\b" + name + "\\b").test(attr(elem, "class"));
}

function attr(elem, name, value) {
  if (typeof value !== "undefined") {
    elem.setAttribute(name, value);
  } else {
    return elem.getAttribute(name) || "";
  }
}

function on(elem, type, listener, useCapture = false) {
  elem.addEventListener(type, listener, useCapture);
}

function off(elem, type, listener, useCapture = false) {
  elem.removeEventListener(type, listener, useCapture);
}
</script>
