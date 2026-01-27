<!--
 * @Description: 视频消息布局
 * @Author: broccoli
 * @LastEditors: broccoli
-->
<template>
  <div>
    <div class="video-item">
      <span class="video-item-duration">{{ item.video.play_length }}'</span>
      <i class="el-icon-video-play" @click="play(item.video.attachment,'video')" />
    </div>
    <VideoModal v-show="dia" ref="videoModal" @closeModal="closeModal" />
  </div>

</template>

<script>
import 'video.js/dist/video-js.css';
import 'vue-video-player/src/custom-theme.css';
import VideoModal from './videoModal.vue';

export default {
  components: { VideoModal },
  props: {
    item: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      dia: false
    };
  },
  mounted() {},
  methods: {
    play(url) {
      if (url) {
        this.dia = true;
        this.$refs.videoModal.play(url);
      } else {
        this.msgError('文件不存在！');
      }
    },
    /**
     * 关闭弹窗
     */
    closeModal() {
      this.dia = false;
      this.$refs.videoModal.pause();
    }
  }
};

</script>
<style lang="scss" scoped>
.video-item {
  display: flex;
  align-items: center;
  .el-icon-video-play {
    font-size: 40px;
    color: #199ed8;
    cursor: pointer;
    margin-left: 10px;
  }
}
</style>
