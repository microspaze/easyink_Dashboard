<!--
 * @Description: 语音消息布局
 * @Author: broccoli
 * @LastEditors: broccoli
-->
<template>
  <div class="voice-item" style="display:flex;align-items:center;">
    <div v-if="item.msgType=='voice'" style="display:flex;align-items:center;margin-left:10px">
      <el-slider v-model="progress" style="width:175px;" :min="0" :max="item.voice.play_length" :step="0.1" :disabled="voicePlayObj[item.msgId] && voicePlayObj[item.msgId].isPlaying" @change="progressChange" />
      <label for="amr-progress" style="margin:0 10px;min-width:70px;text-align: center;">
        <span class="amr-cur">{{ progress }}'</span>
        <span> / </span>
        <span class="amr-duration">{{ item.voice.play_length }}'</span>
      </label>
      <img v-if="voicePlayObj[item.msgId] && voicePlayObj[item.msgId].isPlaying" :src="dealVoiceImg(true)" @click="pauseVoice(item)">
      <img v-else :src="dealVoiceImg(false)" @click="playVoice(item)">
    </div>
    <div v-if="item.msgType=='meeting_voice_call'" style="display:flex;align-items:center;height:40px;overflow:hidden;">
      <audio controls :src="item.meetingVoiceCall.attachment" type="audio/mpeg" />
    </div>
  </div>
</template>

<script>
import 'video.js/dist/video-js.css';
import 'vue-video-player/src/custom-theme.css';
import BenzAMRRecorder from 'benz-amr-recorder';

const PROGRESS_UPDATE_INTERVAL = 100;
const PROGRESS_DELTA_DURATION = 0.1;

export default {
  props: {
    item: {
      type: Object,
      default: () => {}
    }
  },
  data() {
    return {
      progress: 0,
      voicePlayObj: {}
    };
  },
  mounted() {},
  methods: {
    dealVoiceImg(isPlaying) {
      if (isPlaying) {
        return require('@/assets/image/voice-left.gif');
      } else {
        return require('@/assets/image/voice-left.png');
      }
    },
    playVoice(msg) {
      const attachment = msg.voice.attachment;
      if (attachment) {
        this.urlToBlob(attachment, msg);
      } else {
        this.msgError('文件不存在！');
      }
    },
    pauseVoice(msg) {
      const msgId = msg.msgId;
      const amr = this.voicePlayObj[msgId].amr;
      amr.pause();
      this.changeMsgVoiceStatus(msgId, false);
    },
    changeMsgVoiceStatus(msgId, status) {
      this.voicePlayObj[msgId].isPlaying = status;
      // 产生一个新的对象，让页面重新渲染。
      this.voicePlayObj = { ...this.voicePlayObj };
    },
    urlToBlob(file_url, msg) {
      const xhr = new XMLHttpRequest();
      const this_ = this;
      xhr.open('get', file_url, true);
      xhr.responseType = 'blob';
      xhr.onload = function() {
        const blob = this.response;
        const file = new File([blob], '语音.amr', { type: 'audio/amr' });
        this_.playOrResumeAudio(file, msg);
      };
      xhr.send();
    },
    progressChange(value) {
      const this_ = this;
      this_.progress = value;
    },
    playOrResumeAudio(file, msg) {
      const this_ = this;
      const msgId = msg.msgId;
      if (this_.progress + PROGRESS_DELTA_DURATION >= msg.voice.play_length) {
        this_.progress = 0;
        this_.changeMsgVoiceStatus(msgId, false);
      }

      // 第一次点击语音
      if (!this.voicePlayObj[msgId]) {
        const amr = new BenzAMRRecorder();
        try {
          amr.initWithBlob(file).then(function() {
            this_.voicePlayObj[msgId] = {};
            this_.voicePlayObj[msgId].amr = amr;
            amr.play();
            setInterval(function() {
              if (amr.isPlaying()) {
                this_.progress = Number.parseFloat(amr.getCurrentPosition().toFixed(1));
              }
            }, PROGRESS_UPDATE_INTERVAL);
            this_.changeMsgVoiceStatus(msgId, true);
            amr.onEnded(function() {
              this_.changeMsgVoiceStatus(msgId, false);
            });
          });
        } catch (err) {
          console.error('语音初始化失败', err);
        }
      } else {
        const amr = this_.voicePlayObj[msgId].amr;
        if (this_.progress) {
          amr.setPosition(this_.progress);
        }
        amr.playOrResume();
        this_.changeMsgVoiceStatus(msgId, true);
        amr.onEnded(function() {
          this_.changeMsgVoiceStatus(msgId, false);
        });
      }
    }
  }
};

</script>
<style lang="scss" scoped>

</style>
