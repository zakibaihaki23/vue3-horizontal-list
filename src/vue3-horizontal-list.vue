<template>
  <div class="vue-horizontal-list" ref="container">
    <div class="vhl-navigation" v-if="width.window > options.navigation.start">
      <div @click="prev" v-if="hasPrev" class="vhl-btn-left">
        <slot name="nav-prev">
          <svg
            :fill="options.navigation.color"
            width="32px"
            height="32px"
            viewBox="0 0 24 24"
          >
            <path
              d="M10.757 12l4.95 4.95a1 1 0 1 1-1.414 1.414l-5.657-5.657a1 1 0 0 1 0-1.414l5.657-5.657a1 1 0 0 1 1.414 1.414L10.757 12z"
            />
          </svg>
        </slot>
      </div>

      <div @click="next" v-if="hasNext" class="vhl-btn-right">
        <slot name="nav-next">
          <svg
            :fill="options.navigation.color"
            width="32px"
            height="32px"
            viewBox="0 0 24 24"
          >
            <path
              d="M13.314 12.071l-4.95-4.95a1 1 0 0 1 1.414-1.414l5.657 5.657a1 1 0 0 1 0 1.414l-5.657 5.657a1 1 0 0 1-1.414-1.414l4.95-4.95z"
            />
          </svg>
        </slot>
      </div>
    </div>

    <div class="vhl-container" :style="style.container">
      <div
        class="vhl-list"
        ref="list"
        :class="options.list.class"
        :style="style.list"
        @scroll="scrollHandler"
      >
        <div
          v-for="(item, index) in dataItems"
          ref="item"
          class="vhl-item"
          :class="options.item.class"
          :style="style.item"
          :key="index"
        >
          <slot v-if="item.type === 'start'" name="start"></slot>
          <slot v-else-if="item.type === 'end'" name="end"></slot>
          <slot v-else-if="item.type === 'item'" v-bind:item="item.item">{{
            item
          }}</slot>
        </div>

        <div :style="style.tail"></div>
      </div>
      <div class="vhl-pagination">
        <span
          v-for="(item, index) in paginationDots"
          :key="index"
          @click="goToPage(index)"
          :class="{ 'active': isActiveDot(index) }"
        ></span>
      </div>
    </div>
  </div>
</template>

<script>
import {
  ref,
  reactive,
  onMounted,
  computed,
  nextTick,
  useSlots,
  onBeforeUnmount,
} from "vue";

export default {
  name: "vue3HorizontalList",
  props: {
    items: {
      type: Array,
      required: true,
    },
    options: {
      type: Object,
      required: false,
    },
  },
  setup(props) {
    const position = ref(0);
    const width = reactive({
      container: 0,
      window: 576,
    });

    const container = ref(null);
    const list = ref(null);
    const item = ref(null);

    const $resize = () => {
      width.window = window.innerWidth;
      width.container = container.value.clientWidth;
    };

    const scrollTimer = ref(null);
    const autoPlayInterval = ref(null);

    const dataItems = computed(() => {
      const slots = useSlots();
      const start = slots.start;
      const end = slots.end;
      return [
        ...(start ? [{ type: "start" }] : []),
        ...props.items.map((value) => ({ type: "item", item: value })),
        ...(end ? [{ type: "end" }] : []),
      ];
    });

    const length = computed(() => dataItems.value.length);

    const options = computed(() => {
      return {
        navigation: {
          start: props.options?.navigation?.start ?? 992,
          color: props.options?.navigation?.color ?? "#000",
        },
        item: {
          class: props.options?.item?.class ?? "",
          padding: props.options?.item?.padding ?? 16,
        },
        list: {
          class: props.options?.list?.class ?? "",
          windowed: props.options?.list?.windowed ?? 1200,
          padding: props.options?.list?.padding ?? 24,
        },
        responsive: [
          ...(props.options?.responsive ?? []),
          { end: 576, size: 1 },
          { start: 576, end: 768, size: 2 },
          { start: 768, end: 992, size: 3 },
          { start: 992, end: 1200, size: 4 },
          { start: 1200, size: 5 },
        ],
        position: {
          start: props.options?.position?.start ?? 0,
        },
        autoplay: {
          play: props.options?.autoplay?.play ?? false,
          speed: props.options?.autoplay?.speed ?? 2000,
          repeat: props.options?.autoplay?.repeat ?? false,
        },
      };
    });

    const style = computed(() => {
      const style = {
        container: {
          marginLeft: "",
          marginRight: "",
        },
        list: {},
        item: {
          width: "",
          paddingLeft: "",
          paddingRight: "",
          marginRight: "",
        },
        tail: {},
      };

      if (width.window < options.value.list.windowed) {
        style.container.marginLeft = `-${options.value.list.padding}px`;
        style.container.marginRight = `-${options.value.list.padding}px`;

        style.item.width = `${(workingWidth.value - (size.value - 1) * options.value.item.padding) / size.value}px`;
        style.item.paddingLeft = `${options.value.list.padding}px`;
        style.item.paddingRight = `${options.value.item.padding}px`;
        style.item.marginRight = `-${options.value.list.padding}px`;
      } else {
        style.item.paddingLeft = `${options.value.item.padding / 2}px`;
        style.item.paddingRight = `${options.value.item.padding / 2}px`;

        style.container.marginLeft = `-${options.value.item.padding / 2}px`;
        style.container.marginRight = `-${options.value.item.padding / 2}px`;

        style.item.width = `${(workingWidth.value - (size.value - 1) * options.value.item.padding) / size.value}px`;
      }

      return style;
    });

    const itemWidth = computed(() => (workingWidth.value - (size.value - 1) * options.value.item.padding) / size.value);

    const workingWidth = computed(() => {
      if (width.window < options.value.list.windowed) {
        return width.window - options.value.list.padding * 2;
      } else {
        return width.container;
      }
    });

    const size = computed(() => {
      const width = workingWidth.value;
      return options.value.responsive.find((value) => {
        return (
          (!value.start || value.start <= width) &&
          (!value.end || value.end >= width)
        );
      }).size;
    });

    const totalPages = computed(() => Math.ceil(length.value / size.value));

    const hasNext = computed(() => position.value < totalPages.value - 1);
    const hasPrev = computed(() => position.value > 0);

    const go = (positionTo) => {
      position.value = Math.max(0, Math.min(positionTo, totalPages.value - 1));
      const left = (itemWidth.value + options.value.item.padding) * position.value * size.value;
      list.value.scrollTo({ top: 0, left: left, behavior: "smooth" });
    };

    const runAutoPlay = () => {
      autoPlayInterval.value = setInterval(() => {
        if (options.value.autoplay.repeat && position.value === totalPages.value - 1) {
          go(0);
        } else {
          go(position.value + 1);
        }
      }, options.value.autoplay.speed);
    };

    const stopAutoPlay = () => {
      if (autoPlayInterval.value) {
        clearInterval(autoPlayInterval.value);
      }
    };

    const prev = () => {
      if (hasPrev.value) {
        go(position.value - 1);
      }
    };

    const next = () => {
      if (hasNext.value) {
        go(position.value + 1);
      }
    };

    const scrollHandler = () => {
      clearTimeout(scrollTimer.value);

      scrollTimer.value = setTimeout(() => {
        const parentLeftOffset = list.value.getBoundingClientRect().left;
        let closestIndex = 0;
        let smallestDiff = Infinity;

        dataItems.value.forEach((_, index) => {
          const itemElement = item.value[index];
          if (itemElement) {
            const itemLeftOffset = itemElement.getBoundingClientRect().left;
            const diff = Math.abs(itemLeftOffset - parentLeftOffset);

            if (diff < smallestDiff) {
              smallestDiff = diff;
              closestIndex = index;
            }
          }
        });

        position.value = Math.floor(closestIndex / size.value);
      }, 50);
    };

    onMounted(() => {
      require("smoothscroll-polyfill").polyfill();

      $resize();
      window.addEventListener("resize", $resize);

      if (options.value.position.start) {
        nextTick(() => {
          go(options.value.position.start);
        });
      }

      if (options.value.autoplay.play) {
        runAutoPlay();
      }
    });

    onBeforeUnmount(() => {
      window.removeEventListener("resize", $resize);
      stopAutoPlay();
    });

    const paginationDots = computed(() => Array.from({ length: totalPages.value }, (_, index) => index));

    const goToPage = (index) => {
      go(index);
    };

    const isActiveDot = (index) => Math.floor(position.value) === index;

    return {
      position,
      width,
      container,
      list,
      item,
      scrollTimer,
      autoPlayInterval,
      dataItems,
      options,
      style,
      hasPrev,
      hasNext,
      prev,
      next,
      scrollHandler,
      paginationDots,
      goToPage,
      size,
      isActiveDot,
    };
  },
};
</script>

<style scoped>
.vue-horizontal-list {
  position: relative;
}

.vhl-navigation {
  display: flex;
  align-items: center;
  position: absolute;
  width: 100%;
  height: 100%;

  margin-top: -6px;
}

.vhl-btn-left,
.vhl-btn-right {
  width: 48px;
  height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;

  z-index: 2;
}

.vhl-btn-left:hover,
.vhl-btn-right:hover {
  cursor: pointer;
}

.vhl-btn-left {
  margin-left: -50px;
  margin-right: auto;
}

.vhl-btn-right {
  margin-left: auto;
  margin-right: -50px;
}

.vhl-container {
  overflow-y: hidden;
  height: 100%;
  margin-bottom: -24px;
}

.vhl-list {
  display: flex;

  overflow-x: scroll;
  overflow-y: hidden;
  scroll-behavior: smooth;
  -webkit-overflow-scrolling: touch;
  scroll-snap-type: x mandatory;
  scrollbar-width: none;
}

.vhl-list::-webkit-scrollbar {
  width: 0; /* Hide scrollbar for Chrome, Safari, and Opera */
}

.vhl-item {
  box-sizing: content-box;

  padding-top: 24px;
  padding-bottom: 24px;

  z-index: 1;

  /* Prevent content from collapsing when empty. E.g. image while loading height=0. */
  min-height: 1px;
}

.vhl-list > * {
  scroll-snap-align: start;
  flex-shrink: 0;
}

.vhl-pagination {
  display: flex;
  justify-content: center;
  margin-top: 16px;
}

.vhl-pagination span {
  width: 8px;
  height: 8px;
  border-radius: 50%; /* Change the value to create a pill shape */
  background-color: #ccc; /* Default color for inactive dots */
  margin: 0 4px;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.vhl-pagination span.active {
  width: 30px;
  background-color: #0284C7; /* Color for active dot */
  border-radius: 12px;
}
</style>
