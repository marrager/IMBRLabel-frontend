<template>
  <div ref="papayaContainer" class="papaya"></div>
</template>

<script>
import { nextTick } from 'vue';

export default {
  name: 'PapayaViewer',
  mounted() {
    nextTick(() => {
      if (window.papaya && window.papaya.Viewer) {
        // 初始化配置
        const params = {
          images: ["data/myImage.nii.gz"],
          worldSpace: true,
          imageTools: true
        };

        // 初始化Papaya Viewer
        window.papaya.Viewer.reset();  // 确保 Viewer 已加载
        window.papaya.Viewer.setViewerParams(params);
        window.papaya.Viewer.start(this.$refs.papayaContainer);
      } else {
        console.error("Papaya or Viewer is not loaded yet.");
      }
    });
  },
  beforeUnmount() {
    if (window.papaya && window.papaya.Viewer) {
      window.papaya.Viewer.reset();
    }
  }
};

</script>

<style scoped>
.papaya {
  width: 100%;
  height: 600px;
  /* 根据需求调整高度 */
}
</style>
