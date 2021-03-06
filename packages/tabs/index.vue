<template>
  <div class="van-tabs" :class="[`van-tabs--${type}`]">
    <div
      ref="wrap"
      class="van-tabs__wrap"
      :class="[`van-tabs__wrap--${position}`, {
        'van-tabs--scrollable': scrollable,
        'van-hairline--top-bottom': type === 'line'
      }]"
    >
      <div class="van-tabs__nav" :class="`van-tabs__nav--${type}`" ref="nav">
        <div v-if="type === 'line'" class="van-tabs__nav-bar" :style="navBarStyle" />
        <div
          v-for="(tab, index) in tabs"
          :key="index"
          ref="tabs"
          class="van-tab"
          :class="{
            'van-tab--active': index === curActive,
            'van-tab--disabled': tab.disabled
          }"
          @click="onClick(index)"
        >
          <span>{{ tab.title }}</span>
        </div>
      </div>
    </div>
    <div class="van-tabs__content">
      <slot />
    </div>
  </div>
</template>

<script>
import { create } from '../utils';
import { raf } from '../utils/raf';
import { on, off } from '../utils/event';
import scrollUtils from '../utils/scroll';

export default create({
  name: 'van-tabs',

  props: {
    sticky: Boolean,
    active: {
      type: [Number, String],
      default: 0
    },
    type: {
      type: String,
      default: 'line'
    },
    duration: {
      type: Number,
      default: 0.2
    },
    swipeThreshold: {
      type: Number,
      default: 4
    }
  },

  data() {
    return {
      tabs: [],
      position: 'content-top',
      curActive: 0,
      navBarStyle: {}
    };
  },

  watch: {
    active(val) {
      this.correctActive(val);
    },

    tabs(tabs) {
      this.correctActive(this.curActive || this.active);
      this.setNavBar();
    },

    curActive() {
      this.scrollIntoView();
      this.setNavBar();

      // scroll to correct position
      if (this.position === 'page-top' || this.position === 'content-bottom') {
        scrollUtils.setScrollTop(this.scrollEl, scrollUtils.getElementTop(this.$el));
      }
    },

    sticky(isSticky) {
      this.scrollHandler(isSticky);
    }
  },

  mounted() {
    this.correctActive(this.active);
    this.setNavBar();

    this.$nextTick(() => {
      if (this.sticky) {
        this.scrollHandler(true);
      }
      this.scrollIntoView();
    });
  },

  beforeDestroy() {
    /* istanbul ignore next */
    if (this.sticky) {
      this.scrollHandler(false);
    }
  },

  computed: {
    // whether the nav is scrollable
    scrollable() {
      return this.tabs.length > this.swipeThreshold;
    }
  },

  methods: {
    // whether to bind sticky listener
    scrollHandler(init) {
      this.scrollEl = this.scrollEl || scrollUtils.getScrollEventTarget(this.$el);
      (init ? on : off)(this.scrollEl, 'scroll', this.onScroll, true);
      if (init) {
        this.onScroll();
      }
    },

    // adjust tab position
    onScroll() {
      const scrollTop = scrollUtils.getScrollTop(this.scrollEl);
      const elTopToPageTop = scrollUtils.getElementTop(this.$el);
      const elBottomToPageTop = elTopToPageTop + this.$el.offsetHeight - this.$refs.wrap.offsetHeight;
      if (scrollTop > elBottomToPageTop) {
        this.position = 'content-bottom';
      } else if (scrollTop > elTopToPageTop) {
        this.position = 'page-top';
      } else {
        this.position = 'content-top';
      }
    },

    // update nav bar style
    setNavBar() {
      this.$nextTick(() => {
        if (!this.$refs.tabs) {
          return;
        }

        const tab = this.$refs.tabs[this.curActive];
        this.navBarStyle = {
          width: `${tab.offsetWidth || 0}px`,
          transform: `translate(${tab.offsetLeft || 0}px, 0)`,
          transitionDuration: `${this.duration}s`
        };
      });
    },

    // correct the value of active
    correctActive(active) {
      active = +active;
      const exist = this.tabs.some(tab => tab.index === active);
      const defaultActive = (this.tabs[0] || {}).index || 0;
      this.curActive = exist ? active : defaultActive;
    },

    // emit event when clicked
    onClick(index) {
      if (this.tabs[index].disabled) {
        this.$emit('disabled', index);
      } else {
        this.$emit('click', index);
        this.curActive = index;
      }
    },

    // scroll active tab into view
    scrollIntoView() {
      if (!this.scrollable || !this.$refs.tabs) {
        return;
      }

      const tab = this.$refs.tabs[this.curActive];
      const { nav } = this.$refs;
      const { scrollLeft, offsetWidth: navWidth } = nav;
      const { offsetLeft, offsetWidth: tabWidth } = tab;

      this.scrollTo(nav, scrollLeft, offsetLeft - (navWidth - tabWidth) / 2);
    },

    // animate the scrollLeft of nav
    scrollTo(el, from, to) {
      let count = 0;
      const frames = Math.round(this.duration * 1000 / 16);
      const animate = () => {
        el.scrollLeft += (to - from) / frames;
        /* istanbul ignore next */
        if (++count < frames) {
          raf(animate);
        }
      };
      animate();
    }
  }
});
</script>
