<template>
  <div class="player-card-vue">
    <h1 style="margin-top: 0;">{{ videoTitle }}</h1>
    <div id="dplayer-container">
      <div v-if="showPlaceholder" id="video-placeholder-vue">
        <div class="icon">▶️</div>
        <div>请先选择本地视频文件</div>
      </div>
      <!-- DPlayer will be mounted here by its script -->
    </div>

    <div v-if="isLoadingDanmaku" class="progress-container-vue" id="progressContainerVue">
      <progress id="loadProgressVue" :value="danmakuLoadProgress" max="100"></progress>
      <div id="progressTextVue">{{ danmakuLoadText }}</div>
    </div>
    <div class="status-vue" id="statusVue">{{ statusMessage }}</div>
  </div>
</template>

<script>
export default {
  name: 'PlayerCard',
  props: {
    videoTitle: {
      type: String,
      default: '本地弹幕播放器'
    },
    showPlaceholder: {
      type: Boolean,
      default: true
    },
    isLoadingDanmaku: {
      type: Boolean,
      default: false
    },
    danmakuLoadProgress: {
      type: Number,
      default: 0
    },
    danmakuLoadText: {
      type: String,
      default: '加载中: 0%'
    },
    statusMessage: {
      type: String,
      default: '等待加载视频和弹幕...'
    }
  },
  // DPlayer initialization logic will go into methods and mounted hook later
}
</script>

<style scoped>
/* Styles specific to PlayerCard, adapted from original style.css */
.player-card-vue {
    flex: 0 0 auto;
    background: #fff;
    border-radius: 28px;
    box-shadow: 0 6px 32px 0 rgba(30, 34, 40, 0.10);
    padding: 32px 32px 24px 32px;
    display: flex;
    flex-direction: column;
    align-items: center;
    width: 750px; /* Default width, will be overridden by main-layout on smaller screens */
    height: fit-content;
}

@media (max-width: 900px) {
    .player-card-vue {
        max-width: none;
        width: 100%;
        margin: 0;
        padding: 24px 20px;
    }
}
@media (max-width: 480px) {
    .player-card-vue {
        padding: 16px 16px 20px 16px;
    }
}


#dplayer-container { /* Changed from #dplayer to avoid conflicts if original script is loaded */
    width: 100%;
    border-radius: 20px;
    overflow: hidden;
    box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.06);
    margin-bottom: 32px;
    position: relative;
    aspect-ratio: 16/9;
    max-width: 720px;
}

#dplayer-container .dplayer-video-wrap { /* Ensure DPlayer internals fill container */
    width: 100%;
    height: 100%;
}

#video-placeholder-vue {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    color: #bbb;
    font-size: 22px;
    background: #f7f8fa;
    z-index: 2;
    pointer-events: none;
    height: 100%;
    width: 100%;
}

#video-placeholder-vue .icon {
    font-size: 60px;
    margin-bottom: 18px;
    color: #e3e5e7;
}

.status-vue {
    padding: 14px;
    background: #f2f3f7;
    border-radius: 14px;
    color: #555;
    font-size: 16px;
    text-align: center;
    width: 100%; /* Ensure it takes full width within the card */
    box-sizing: border-box; /* Include padding in width calculation */
}

.progress-container-vue {
    margin-top: 18px;
    width: 100%; /* Ensure it takes full width */
    display: block; /* Changed from none, visibility controlled by v-if */
}

.progress-container-vue progress {
    width: 100%;
    height: 18px;
    border-radius: 9px;
    overflow: hidden;
    background: #e3e5e7;
    accent-color: #00A1D6;
}

#progressTextVue {
    color: #00A1D6;
    font-weight: 600;
    margin-top: 6px;
    font-size: 16px;
    text-align: center;
}
</style>
