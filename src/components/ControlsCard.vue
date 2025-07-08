<template>
  <div class="controls-card-vue">
    <img class="avatar-vue" src="../assets/favicon.png" alt="favicon">
    <div style="text-align: center; margin: 15px 0; display: flex; justify-content: center; gap: 8px;">
      <a href="https://github.com/hexwarrior6/DanMuWebPlayer" target="_blank"
          class="external-link-btn github-btn">
        ⭐ Star on GitHub
      </a>
      <a href="https://docs.qq.com/aio/p/sch8dr6ly9ajte9?p=NSA0My9Rm3rACf4atrT2nH" target="_blank"
          class="external-link-btn docs-btn">
        📄 说明文档
      </a>
    </div>
    <div class="info-detail-vue">本地弹幕播放器，支持本地视频、m3u8直播流与弹幕文件播放。</div>

    <div class="info-section-vue">
      <b>使用方法：</b>
      <ul class="usage-list-vue">
        <li>选择本地视频和弹幕文件</li>
        <li>或输入m3u8直播源地址</li>
        <li>播放视频/直播即可实时显示弹幕</li>
      </ul>
    </div>

    <!-- Danmaku Speed Control -->
    <div class="info-section-vue">
      <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 8px;">
        <b>弹幕速度：</b>
        <span id="speedValueVue">{{ localDanmakuSpeed.toFixed(1) }}</span>
      </div>
      <input type="range" id="speedSliderVue" min="0.1" max="2" step="0.1" :value="localDanmakuSpeed" @input="updateDanmakuSpeed">
    </div>

    <!-- Video Source Section -->
    <div class="info-section-vue">
      <b>视频源(任选一种)：</b>
      <div class="file-control-group-vue">
        <label for="videoFileVue" class="file-label-vue">{{ videoFileLabel }}</label>
        <input type="file" id="videoFileVue" class="file-input-vue" accept="video/*" @change="handleVideoFileChange">
      </div>
      <div class="input-group-container-vue">
        <input type="text" id="liveStreamInputVue" placeholder="m3u8直播源地址" class="file-input-style-vue" v-model="liveStreamUrl" @keyup.enter="loadLiveStream">
        <button id="loadLiveStreamBtnVue" class="danmu-fetch-btn-vue" @click="loadLiveStream">加载</button>
      </div>
    </div>

    <!-- Danmaku Source Section -->
    <div class="info-section-vue">
      <b>弹幕源(任选一种)：</b>
      <div class="file-control-group-vue">
        <label for="danmuFileXmlVue" class="file-label-vue">{{ xmlFileLabel }}</label>
        <input type="file" id="danmuFileXmlVue" class="file-input-vue" accept=".xml" @change="handleDanmakuFileChange('xml', $event)">

        <label for="danmuFileJsonVue" class="file-label-vue">{{ jsonFileLabel }}</label>
        <input type="file" id="danmuFileJsonVue" class="file-input-vue" accept=".json" @change="handleDanmakuFileChange('json', $event)">

        <div class="input-group-container-vue">
          <input type="text" id="rrmjDanmuIdInputVue" placeholder="输入人人视频弹幕ID" class="file-input-style-vue" v-model="rrmjDanmuId" @keyup.enter="fetchRrmjDanmaku">
          <button id="rrmjDanmuFetchBtnVue" class="danmu-fetch-btn-vue" @click="fetchRrmjDanmaku">获取</button> <!-- Corrected class binding -->
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'ControlsCard',
  props: {
    initialDanmakuSpeed: {
      type: Number,
      default: 1.0
    },
    xmlFileNameHint: { // From App.vue, for resetting label
        type: String,
        default: "标准XML弹幕"
    },
    jsonFileNameHint: { // From App.vue
        type: String,
        default: "人人视频JSON弹幕"
    }
  },
  data() {
    return {
      localDanmakuSpeed: this.initialDanmakuSpeed,
      liveStreamUrl: '',
      rrmjDanmuId: '',
      videoFileLabel: '选择本地视频文件',
      xmlFileLabel: this.xmlFileNameHint, // Initialize from prop
      jsonFileLabel: this.jsonFileNameHint, // Initialize from prop
    };
  },
  watch: {
    initialDanmakuSpeed(newVal) {
      this.localDanmakuSpeed = newVal;
    },
    xmlFileNameHint(newVal) {
        this.xmlFileLabel = newVal;
    },
    jsonFileNameHint(newVal) {
        this.jsonFileLabel = newVal;
    }
  },
  methods: {
    updateDanmakuSpeed(event) {
      this.localDanmakuSpeed = parseFloat(event.target.value);
      this.$emit('danmaku-speed-changed', this.localDanmakuSpeed);
    },
    handleVideoFileChange(event) {
      const file = event.target.files[0];
      if (file) {
        this.videoFileLabel = `视频: ${file.name}`;
        this.$emit('video-selected', { file });
        this.liveStreamUrl = ''; // Clear live stream input
      } else {
        this.videoFileLabel = '选择本地视频文件';
      }
    },
    loadLiveStream() {
      const url = this.liveStreamUrl.trim();
      if (url) {
         // Basic m3u8 validation
        if (!url.toLowerCase().endsWith('.m3u8') && !url.toLowerCase().includes('.m3u8?')) {
            alert('请输入有效的m3u8格式直播源地址'); // Or emit a status update
            // Optionally emit a status to App.vue to display in statusMessage
            this.$emit('status-update', '错误：请输入有效的m3u8格式直播源地址');
            return;
        }
        this.$emit('video-selected', { url: url, isLive: true });
        this.videoFileLabel = '选择本地视频文件'; // Reset video file label
        // Clear file input visually if possible (resetting value of input type=file is tricky)
        const videoFileInput = document.getElementById('videoFileVue');
        if (videoFileInput) videoFileInput.value = '';

      }
    },
    handleDanmakuFileChange(type, event) { // Pass event directly
      const file = event.target.files[0];
      if (file) {
        if (type === 'xml') {
          this.xmlFileLabel = `XML: ${file.name}`;
          this.jsonFileLabel = this.jsonFileNameHint; // Reset other danmaku input label
           const jsonFileInput = document.getElementById('danmuFileJsonVue');
           if (jsonFileInput) jsonFileInput.value = '';
        } else if (type === 'json') {
          this.jsonFileLabel = `JSON: ${file.name}`;
          this.xmlFileLabel = this.xmlFileNameHint; // Reset other danmaku input label
          const xmlFileInput = document.getElementById('danmuFileXmlVue');
          if (xmlFileInput) xmlFileInput.value = '';
        }
        this.$emit('danmaku-selected', { file, type });
        this.rrmjDanmuId = ''; // Clear RRSP ID input
      } else {
        // Reset to default if no file is chosen (e.g., user cancels file dialog)
        if (type === 'xml') this.xmlFileLabel = this.xmlFileNameHint;
        if (type === 'json') this.jsonFileLabel = this.jsonFileNameHint;
      }
    },
    fetchRrmjDanmaku() {
      const id = this.rrmjDanmuId.trim();
      if (id) {
        this.$emit('danmaku-selected', { rrspId: id });
        // Reset file input labels and their actual input values
        this.xmlFileLabel = this.xmlFileNameHint;
        this.jsonFileLabel = this.jsonFileNameHint;
        const xmlFileInput = document.getElementById('danmuFileXmlVue');
        if (xmlFileInput) xmlFileInput.value = '';
        const jsonFileInput = document.getElementById('danmuFileJsonVue');
        if (jsonFileInput) jsonFileInput.value = '';
      }
    }
  }
}
</script>

<style scoped>
/* Styles specific to ControlsCard, adapted from original style.css */
.controls-card-vue {
    flex: 0 0 340px;
    background: linear-gradient(135deg, #fafdff 60%, #e9f3ff 100%);
    border-radius: 32px;
    box-shadow: 0 8px 36px 0 rgba(30, 34, 40, 0.13), 0 1.5px 6px 0 rgba(0, 161, 214, 0.06);
    padding: 20px;
    display: flex;
    flex-direction: column;
    align-items: center;
    min-width: 260px;
    max-width: 340px;
    border: 1.5px solid #e3e5e7;
    position: relative;
    overflow: visible;
    height: auto;
    min-height: auto;
}

@media (max-width: 900px) {
    .controls-card-vue {
        flex: none;
        min-width: auto;
        max-width: none;
        width: 100%;
        height: auto;
        min-height: auto;
        overflow: visible;
        padding: 20px 20px 32px 20px;
        margin: 0;
    }
}
@media (max-width: 480px) {
    .controls-card-vue {
        padding: 16px 16px 24px 16px;
        border-radius: 24px;
    }
}

.avatar-vue {
    width: 70px;
    height: 70px;
    border-radius: 50%;
    background: #e3e5e7; /* Fallback color */
    object-fit: cover;
    border: 3px solid #fff;
    box-shadow: 0 2px 12px 0 rgba(0, 161, 214, 0.08);
}
@media (max-width: 900px) {
    .avatar-vue { width: 60px; height: 60px; }
}
@media (max-width: 480px) {
    .avatar-vue { width: 50px; height: 50px; }
}

.external-link-btn {
    padding: 8px 16px;
    color: white;
    text-decoration: none;
    border-radius: 8px;
    font-size: 14px; /* Adjusted for consistency */
    font-weight: 500; /* Adjusted */
}
.github-btn {
    background-color: #24292e;
}
.docs-btn {
    background-color: #0078d4;
}


.info-detail-vue {
    font-size: 15px;
    color: #4a6fa1;
    text-align: center;
    background: #f2f7fb;
    border-radius: 10px;
    padding: 6px 10px;
    width: calc(100% - 20px); /* Ensure padding is accounted for */
    box-sizing: border-box;
}
@media (max-width: 900px) {
    .info-detail-vue { font-size: 14px; padding: 5px 8px; }
}
@media (max-width: 480px) {
    .info-detail-vue { font-size: 13px; padding: 4px 6px; }
}


.info-section-vue {
    width: 100%;
    margin-bottom: 8px;
    margin-top: 12px;
    position: relative;
}

.info-section-vue:not(:first-child) {
    margin-top: 16px;
}

.info-section-vue:not(:first-child)::before {
    content: "";
    display: block;
    width: 60%;
    height: 1px;
    background: linear-gradient(90deg, #e3e5e7 0%, #b8c0cc 100%);
    margin: 0 auto 12px auto;
    border-radius: 1px;
}

.info-section-vue b {
    display: block;
    text-align: left;
    font-size: 16px;
    color: #333;
    margin-bottom: 8px;
    margin-left: 20px;
}
@media (max-width: 900px) {
    .info-section-vue b { font-size: 15px; margin-left: 16px; margin-bottom: 6px; }
}
@media (max-width: 480px) {
    .info-section-vue b { font-size: 14px; margin-left: 12px; }
}

.usage-list-vue {
    padding-left: 0;
    margin: 8px 0 0 0;
    color: #3a4a5a;
    font-size: 14px;
    list-style: none;
    text-align: center;
}

.usage-list-vue li {
    margin-bottom: 5px;
    text-align: left;
    display: flex;
    margin-left: 20px;
}

.usage-list-vue li::before {
    content: "•";
    color: #00A1D6;
    font-size: 16px;
    margin-right: 6px;
    line-height: 1;
}
@media (max-width: 900px) {
    .usage-list-vue { font-size: 13px; margin: 6px 0 0 0; }
    .usage-list-vue li { margin-left: 16px; margin-bottom: 4px;}
}
@media (max-width: 480px) {
    .usage-list-vue { font-size: 12px; }
    .usage-list-vue li { margin-left: 12px; }
}


/* Danmaku Speed Slider */
#speedSliderVue {
    width: calc(100% - 40px); /* Adjust to fit within padding */
    margin: auto; /* Center it */
    height: 8px;
    border-radius: 4px;
    background: linear-gradient(90deg, #00A1D6 0%, #e3e5e7 100%); /* This might need JS to update fill */
    outline: none;
    accent-color: #00A1D6; /* Modern way to color range inputs */
    display: block; /* Ensure it takes up block space */
}
/* Thumb styles for various browsers */
#speedSliderVue::-webkit-slider-thumb {
    -webkit-appearance: none;
    appearance: none;
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #fff;
    border: 2px solid #00A1D6;
    box-shadow: 0 2px 6px 0 rgba(0, 161, 214, 0.10);
    cursor: pointer;
    transition: background 0.2s;
}
#speedSliderVue::-moz-range-thumb {
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #fff;
    border: 2px solid #00A1D6;
    box-shadow: 0 2px 6px 0 rgba(0, 161, 214, 0.10);
    cursor: pointer;
    transition: background 0.2s;
}
#speedSliderVue::-ms-thumb { /* IE/Edge specific */
    width: 18px;
    height: 18px;
    border-radius: 50%;
    background: #fff;
    border: 2px solid #00A1D6;
    box-shadow: 0 2px 6px 0 rgba(0, 161, 214, 0.10);
    cursor: pointer;
    transition: background 0.2s;
}


#speedValueVue {
    color: #00A1D6;
    font-weight: bold;
    font-size: 17px;
    margin-right: 25px; /* Ensure it's within the flex container for alignment */
}

/* File and Input Groups */
.file-control-group-vue {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 10px;
    align-items: center;
    margin-top: 8px;
    margin-bottom: 10px;
}

.file-label-vue,
.input-group-container-vue {
    display: block;
    width: 100%;
    max-width: 300px;
    margin-left: auto;
    margin-right: auto;
    box-sizing: border-box;
}

.file-label-vue { /* Copied from original button style */
    border: none;
    border-radius: 14px;
    font-size: 18px;
    font-weight: 600;
    padding: 14px 22px;
    background: #222;
    color: #fff;
    cursor: pointer;
    transition: background 0.2s, box-shadow 0.2s;
    box-shadow: 0 2px 8px 0 rgba(0, 0, 0, 0.08);
    text-align: center; /* Ensure text is centered */
}
.file-label-vue:hover {
    background: #444;
}

.file-input-vue {
    display: none; /* Hidden, triggered by label */
}

.input-group-container-vue {
    display: flex;
    gap: 8px;
    height: 48px;
    padding: 0;
}

.file-input-style-vue { /* For text inputs like m3u8 and RRSP ID */
    flex: 1;
    border: 2px solid #e3e5e7;
    border-radius: 14px;
    font-size: 16px;
    outline: none;
    background: #f7f8fa;
    transition: border 0.2s;
    height: 100%;
    box-sizing: border-box;
    padding: 0 12px;
    min-width: 0;
}
.file-input-style-vue:focus {
    border-color: #00A1D6;
}

.danmu-fetch-btn-vue { /* For Load and Get buttons */
    padding: 0 18px;
    font-size: 16px;
    border-radius: 14px;
    font-weight: 600;
    background: #222;
    color: #fff;
    border: none;
    cursor: pointer;
    transition: background 0.2s, box-shadow 0.2s;
    box-shadow: 0 2px 8px 0 rgba(0, 0, 0, 0.08);
    height: 100%;
    width: 80px;
    flex-shrink: 0;
}
.danmu-fetch-btn-vue:hover {
    background: #444;
}

</style>
