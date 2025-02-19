<template>
  <div class="editContainer">
    <Header></Header>
    <div class="content" v-if="showContent">
      <component :is="activeLayout" :key="layout"></component>
    </div>
  </div>
</template>

<script setup>
import Header from "@/components/Header.vue";
import { computed, watch, getCurrentInstance, onUnmounted, ref } from "vue";
import { useStore } from "vuex";
import { layoutMap, defaultViewThemeConfig } from "@/config/constants";
import { useRouter, useRoute } from "vue-router";
import { initMonacoEditor } from "@/utils/monacoEditor";
import nprogress from "nprogress";

// hooks定义部分
const useInit = () => {
  const store = useStore();
  const proxy = getCurrentInstance().proxy;
  const showContent = ref(false);
  const init = async (callback) => {
    // 初始化编辑器
    await initMonacoEditor();
    callback();
    showContent.value = true;
  };
  const route = useRoute();
  // 获取当前编辑数据
  const getData = async () => {
    nprogress.start();
    await store.dispatch("getData", route.params.id);
    proxy.$eventEmitter.emit("reset_code");
    nprogress.done();
  };
  // 监听路由变化
  watch(
    () => {
      return route.params;
    },
    () => {
      getData();
    }
  );
  getData();
  return {
    store,
    showContent,
    proxy,
    router: useRouter(),
    init,
  };
};

// 布局
const useLayout = ({ store }) => {
  const layout = computed(() => {
    return store.state.editData.config.layout;
  });
  const activeLayout = computed(() => {
    return layoutMap[layout.value];
  });
  return {
    layout,
    activeLayout,
  };
};

// 新窗口预览
const useWindowPreview = ({ store, layout, router, proxy }) => {
  // 预览窗口
  let previewWindow = null;
  // 通知预览窗口进行刷新
  const editData = computed(() => store.state.editData);
  const previewWindowRun = () => {
    previewWindow &&
      previewWindow.postMessage({
        type: "preview",
        data: {
          config: {
            openAlmightyConsole: editData.value.config.openAlmightyConsole,
          },
          code: {
            HTML: {
              language: editData.value.code.HTML.language,
              content: editData.value.code.HTML.content,
            },
            CSS: {
              language: editData.value.code.CSS.language,
              content: editData.value.code.CSS.content,
              resources: editData.value.code.CSS.resources.map((item) => {
                return {
                  ...item,
                };
              }),
            },
            JS: {
              language: editData.value.code.JS.language,
              content: editData.value.code.JS.content,
              resources: editData.value.code.JS.resources.map((item) => {
                return {
                  ...item,
                };
              }),
            },
          },
        },
      });
  };
  proxy.$eventEmitter.on("preview_window_run", previewWindowRun);
  onUnmounted(() => {
    proxy.$eventEmitter.off("preview_window_run", previewWindowRun);
  });

  // 创建和销毁预览窗口
  const previewLayoutHandle = () => {
    // 新窗口预览模式
    if (layout.value === "newWindowPreview") {
      if (!previewWindow) {
        let previewUrl = router.resolve({
          name: "Preview",
        });
        previewWindow = window.open(previewUrl.href);
        previewWindow.onload = () => {
          previewWindowRun();
        };
      }
    } else {
      if (previewWindow) {
        previewWindow.close();
        previewWindow = null;
      }
    }
  };
  watch(
    () => {
      return layout.value;
    },
    () => {
      previewLayoutHandle();
    }
  );

  return {
    previewLayoutHandle,
    previewWindowRun,
  };
};

// 界面主题，设置css变量
const useTheme = ({ proxy, store }) => {
  // 主题同步设置
  const pageThemeSyncCodeTheme = computed(() => {
    return store.state.editData.config.pageThemeSyncCodeTheme;
  });
  let lastThemeData = null;
  // 获取变量的值
  const getThemeValue = (item, data) => {
    let arr = defaultViewThemeConfig[item];
    let len = arr.length;
    if (!data || !pageThemeSyncCodeTheme.value) {
      return arr[len - 1];
    }
    for (let i = 0; i < len - 1; i++) {
      let cur = arr[i];
      if (data.colors[cur] !== undefined) {
        return data.colors[cur];
      }
    }
    return arr[len - 1];
  };
  // 更新主题变量
  const updateTheme = (data) => {
    lastThemeData = data;
    Object.keys(defaultViewThemeConfig).forEach((item) => {
      document.documentElement.style.setProperty(
        item,
        getThemeValue(item, data)
      );
    });
  };
  updateTheme();
  watch(pageThemeSyncCodeTheme, () => {
    updateTheme(lastThemeData);
  });
  proxy.$eventEmitter.on("set-theme", updateTheme);
  onUnmounted(() => {
    proxy.$eventEmitter.off("set-theme", updateTheme);
  });
};

// created部分
const { proxy, router, store, init, showContent } = useInit();
const { layout, activeLayout } = useLayout({ store });
const { previewLayoutHandle } = useWindowPreview({
  store,
  layout,
  router,
  proxy,
});
useTheme({ proxy, store });
init(() => {
  previewLayoutHandle();
});
</script>

<style scoped lang="less">
.editContainer {
  position: fixed;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;

  .content {
    width: 100%;
    height: 100%;
    overflow: hidden;
  }
}
</style>
<style lang="less">
#nprogress .bar {
  background: var(--header-btn-color) !important;
}
#nprogress .peg {
  box-shadow: 0 0 10px var(--header-btn-color), 0 0 5px var(--header-btn-color) !important;
}
#nprogress .spinner-icon {
  border-top-color: var(--header-btn-color) !important;
  border-left-color: var(--header-btn-color) !important;
}
</style>