<template>
  <div id="app-container" class="main-layout">
    <PlayerCard
      :videoTitle="videoInfo.title"
      :showPlaceholder="videoInfo.showPlaceholder"
      :isLoadingDanmaku="danmaku.isLoading"
      :danmakuLoadProgress="danmaku.loadProgress"
      :danmakuLoadText="danmaku.loadText"
      :statusMessage="statusMessage"
    />
    <ControlsCard
      @video-selected="handleVideoSelected"
      @danmaku-selected="handleDanmakuSelected"
      @danmaku-speed-changed="handleDanmakuSpeedChanged"
      @status-update="updateStatus"
      :initialDanmakuSpeed="danmaku.speed"
      :xmlFileNameHint="danmaku.xmlFileName"
      :jsonFileNameHint="danmaku.jsonFileName"
    />
  </div>
</template>

<script>
import PlayerCard from './components/PlayerCard.vue';
import ControlsCard from './components/ControlsCard.vue';
import DPlayer from 'dplayer';
import Hls from 'hls.js';

// Make Hls available globally for DPlayer if needed, or pass as custom type
window.Hls = Hls;

export default {
  name: 'App',
  components: {
    PlayerCard,
    ControlsCard
  },
  data() {
    return {
      dp: null, // DPlayer instance
      videoInfo: {
        url: '',
        title: '本地弹幕播放器',
        isLive: false,
        showPlaceholder: true,
      },
      danmaku: {
        list: [],
        isLoading: false,
        loadProgress: 0,
        loadText: '加载中: 0%',
        speed: 1.0,
        // Store file names for display on labels in ControlsCard
        xmlFileName: '标准XML弹幕',
        jsonFileName: '人人视频JSON弹幕',
      },
      statusMessage: '请选择视频文件和弹幕文件或输入直播源',
      // For Web Worker
      danmakuParseWorker: null,
    };
  },
  methods: {
    updateStatus(message) {
      this.statusMessage = message;
      console.log("Status:", message); // For debugging
    },

    initPlayer() {
      if (this.dp) {
        this.dp.destroy();
        this.dp = null;
      }

      if (!this.videoInfo.url) {
        this.updateStatus('视频源未选择');
        this.videoInfo.showPlaceholder = true;
        return;
      }

      this.videoInfo.showPlaceholder = false;
      this.updateStatus('正在初始化播放器...');

      const playerOptions = {
        container: document.getElementById('dplayer-container'), // Make sure this ID is unique in PlayerCard
        autoplay: this.videoInfo.isLive,
        theme: '#FADFA3', // Original theme color
        video: {
          url: this.videoInfo.url,
          type: this.videoInfo.isLive ? 'customHls' : 'auto',
          live: this.videoInfo.isLive,
        },
        danmaku: {
          id: 'vue-local-danmu',
          api: '', // We are loading danmaku manually
          maximum: 1000,
          user: '观众',
          // speedRate: this.danmaku.speed, // DPlayer's own speedRate is different
        },
        contextmenu: [], // Minimal context menu
        customType: {
          customHls: function (video, player) {
            const hls = new Hls();
            hls.loadSource(video.src);
            hls.attachMedia(video);
            hls.on(Hls.Events.MANIFEST_PARSED, () => {
              if (player.options.autoplay) {
                player.play();
              }
            });
          },
        },
      };

      this.dp = new DPlayer(playerOptions);

      if (!this.videoInfo.isLive) {
        this.dp.seek(0);
      }

      this.applyDanmakuSpeedStyle(); // Apply initial speed style

      this.updateStatus(`播放器已初始化。${this.videoInfo.isLive ? '正在加载直播流...' : '等待加载弹幕...'}`);

      if (this.danmaku.list.length > 0 && !this.videoInfo.isLive) {
        this.setupDanmuRealtimeLoader();
        this.updateStatus("播放器和弹幕均已准备就绪");
      } else if (this.danmaku.list.length > 0 && this.videoInfo.isLive) {
        // For live streams, danmaku might be added if API allows, or just shown if pre-loaded
        // Current script doesn't seem to load danmaku for live streams, so we'll stick to that.
         this.updateStatus("播放器准备就绪。直播流不支持加载本地弹幕。");
      }
    },

    handleVideoSelected(payload) {
      this.videoInfo.showPlaceholder = true; // Show placeholder while processing
      if (this.dp) { // Destroy old player first if switching video
          this.dp.destroy();
          this.dp = null;
      }

      if (payload.file) {
        this.videoInfo.isLive = false;
        this.videoInfo.url = URL.createObjectURL(payload.file);
        this.videoInfo.title = payload.file.name;
        this.updateStatus(`已选择本地视频: ${payload.file.name}`);
        // Reset live stream input in ControlsCard if possible (or emit event)
      } else if (payload.url) {
        this.videoInfo.isLive = true;
        this.videoInfo.url = payload.url;
        this.videoInfo.title = '直播流';
        this.updateStatus(`已选择直播源: ${payload.url}`);
        // Reset file input in ControlsCard if possible
      }
      // Defer player initialization until the DOM is updated and #dplayer-container is surely available
      this.$nextTick(() => {
        this.initPlayer();
      });
    },

    handleDanmakuSelected(payload) {
      if (payload.file) {
        this.updateStatus(`正在处理弹幕文件: ${payload.file.name}...`);
        const reader = new FileReader();
        reader.onload = async (e) => {
          try {
            this.danmaku.isLoading = true;
            this.updateStatus("正在解析弹幕(后台处理)...");
            const startTime = performance.now();
            let parsedDanmus = [];

            if (payload.type === 'xml') {
              this.danmaku.xmlFileName = payload.file.name;
              parsedDanmus = await this.parseDanmuXMLWithWorker(e.target.result);
            } else if (payload.type === 'json') {
              this.danmaku.jsonFileName = payload.file.name;
              parsedDanmus = this.parseDanmuJSON(e.target.result); // Assuming JSON parsing is fast
            }

            const endTime = performance.now();
            this.updateStatus(`解析完成，耗时 ${(endTime - startTime).toFixed(2)}ms，共 ${parsedDanmus.length} 条弹幕`);
            this.danmaku.list = parsedDanmus;
            this.danmaku.isLoading = false;

            if (this.dp && !this.videoInfo.isLive) {
              this.setupDanmuRealtimeLoader();
              this.updateStatus("弹幕已加载并准备就绪");
            } else if (this.dp && this.videoInfo.isLive) {
              this.updateStatus('弹幕已解析。直播流不支持显示本地弹幕。');
            } else {
              this.updateStatus('弹幕已解析，请加载视频文件');
            }
          } catch (error) {
            this.updateStatus(`弹幕处理失败: ${error.message}`);
            console.error(error);
            this.danmaku.isLoading = false;
          }
        };
        reader.onerror = () => {
          this.updateStatus('弹幕文件读取失败');
          this.danmaku.isLoading = false;
        };
        reader.readAsText(payload.file);
      } else if (payload.rrspId) {
        this.fetchRrspDanmakuApi(payload.rrspId);
      }
    },

    async fetchRrspDanmakuApi(rrmjId) {
      this.updateStatus('正在从人人视频接口获取弹幕...');
      this.danmaku.isLoading = true;
      try {
        const url = `https://static-dm.rrmj.plus/v1/produce/danmu/EPISODE/${rrmjId}`; // Consider proxy if CORS issues
        const resp = await fetch(url);
        if (!resp.ok) {
          throw new Error(`弹幕接口请求失败: ${resp.status}`);
        }
        const json = await resp.json();
        let danmuArr = [];
        if (Array.isArray(json)) {
          danmuArr = json;
        } else if (json && json.data && Array.isArray(json.data)) {
          danmuArr = json.data;
        } else {
            throw new Error('弹幕接口返回数据格式不正确');
        }

        if (danmuArr.length === 0) {
          this.updateStatus('未获取到弹幕数据');
          this.danmaku.list = [];
        } else {
          const parsedDanmus = this.parseDanmuJSON(JSON.stringify(danmuArr)); // API returns array of objects for parseDanmuJSON
          this.danmaku.list = parsedDanmus;
          this.updateStatus(`成功获取 ${parsedDanmus.length} 条弹幕`);
        }

        this.danmaku.isLoading = false;
        if (this.dp && !this.videoInfo.isLive) {
          this.setupDanmuRealtimeLoader();
        } else if (this.dp && this.videoInfo.isLive) {
          this.updateStatus('弹幕已获取。直播流不支持显示本地弹幕。');
        } else {
          this.updateStatus('弹幕已获取，请加载视频文件');
        }
      } catch (e) {
        this.updateStatus(`获取人人视频弹幕失败: ${e.message}`);
        console.error(e);
        this.danmaku.isLoading = false;
      }
    },

    parseDanmuXMLWithWorker(xmlText) {
      return new Promise((resolve, reject) => {
        if (this.danmakuParseWorker) {
          this.danmakuParseWorker.terminate(); // Terminate existing worker if any
        }
        // Inlined worker code for simplicity
        const workerScript = `
          function parseDanmuXML(xmlText) {
            // ... (regex parsing logic from original script.js)
            const danmuRegex = /<d p="([^"]+)">([^<]+)<\\/d>/g;
            const result = [];
            let match;
            while ((match = danmuRegex.exec(xmlText)) !== null) {
                const attrs = match[1].split(',');
                const text = match[2];
                if (attrs.length >= 8) {
                    const time = parseFloat(attrs[0]);
                    const type = parseInt(attrs[1]); // 1:scroll, 4:bottom, 5:top
                    const color = parseInt(attrs[3]);
                    let danmuType = 'right'; // default scroll
                    if (type === 4) danmuType = 'bottom';
                    else if (type === 5) danmuType = 'top';
                    // Original script only handled type 1, DPlayer supports others
                    // For now, let's stick to original filtering if specific
                     if (type === 1 || type === 4 || type === 5) { // Keep type 1, 4, 5
                        result.push({
                            text: text,
                            color: '#' + color.toString(16).padStart(6, '0'),
                            type: danmuType,
                            time: time
                        });
                    }
                }
            }
            return result;
          }
          self.onmessage = function(e) {
            try {
              const result = parseDanmuXML(e.data);
              self.postMessage({ success: true, data: result });
            } catch (error) {
              self.postMessage({ success: false, error: error.message });
            }
          };
        `;
        const blob = new Blob([workerScript], { type: 'application/javascript' });
        this.danmakuParseWorker = new Worker(URL.createObjectURL(blob));

        this.danmakuParseWorker.onmessage = (e) => {
          if (e.data.success) {
            resolve(e.data.data);
          } else {
            reject(new Error(e.data.error));
          }
          this.danmakuParseWorker.terminate();
          this.danmakuParseWorker = null;
          URL.revokeObjectURL(blob); // Clean up blob URL
        };
        this.danmakuParseWorker.onerror = (error) => {
          reject(error);
          this.danmakuParseWorker.terminate();
          this.danmakuParseWorker = null;
          URL.revokeObjectURL(blob);
        };
        this.danmakuParseWorker.postMessage(xmlText);
      });
    },

    parseDanmuJSON(jsonText) {
      // From original script.js (adapted for DPlayer structure)
      let result = [];
      try {
        const arr = JSON.parse(jsonText);
        for (const item of arr) {
          // Assuming item format is {p: "time,type,size,color,...", d: "text"}
          // or directly the format DPlayer expects: {time, type, color, text}
          if (item.p && item.d) { // RRSP API format with 'p' and 'd'
            const attrs = item.p.split(',');
            if (attrs.length >= 4) { // time, type, size, color
              const time = parseFloat(attrs[0]);
              const type = parseInt(attrs[1]); // 1:scroll, 4:bottom, 5:top
              const colorHex = parseInt(attrs[3]).toString(16).padStart(6, '0');

              let danmuType = 'right'; // default scroll
              if (type === 4) danmuType = 'bottom';
              else if (type === 5) danmuType = 'top';

              // if (type === 1) { // Original script only handled type 1
              result.push({
                  text: item.d,
                  color: `#${colorHex}`,
                  type: danmuType,
                  time: time
              });
              // }
            }
          } else if (item.time !== undefined && item.text !== undefined) { // Direct DPlayer format
             result.push({
                text: item.text,
                color: item.color || '#ffffff',
                type: item.type || 'right',
                time: parseFloat(item.time)
            });
          }
        }
      } catch (e) {
        this.updateStatus('JSON弹幕解析失败: ' + e.message);
        console.error(e);
        return []; // Return empty on error
      }
      return result;
    },

    applyDanmakuSpeedStyle() {
        if (!document) return; // Guard for SSR or non-browser environments
        let speedStyle = document.getElementById('danmu-speed-style-vue');
        if (!speedStyle) {
            speedStyle = document.createElement('style');
            speedStyle.id = 'danmu-speed-style-vue';
            document.head.appendChild(speedStyle);
        }
        const baseDuration = 8.5; // Original base duration
        const newDuration = baseDuration / this.danmaku.speed;
        // This targets DPlayer's own class for danmaku items
        speedStyle.textContent = `.dplayer-danmaku-item { animation-duration: ${newDuration}s !important; transition-duration: ${newDuration}s !important; }`;
    },

    handleDanmakuSpeedChanged(newSpeed) {
      this.danmaku.speed = newSpeed;
      if (this.dp) { // Check if dp is initialized
        // DPlayer has its own internal speed control, but the original script used CSS.
        // We'll stick to the CSS method for now to match original behavior.
        this.applyDanmakuSpeedStyle();
        this.updateStatus(`弹幕速度已调整为: ${newSpeed.toFixed(1)}`);
      }
    },

    setupDanmuRealtimeLoader() {
      if (!this.dp || !this.dp.danmaku || this.videoInfo.isLive) return;

      this.dp.danmaku.clear(); // Clear existing danmaku first
      this.danmaku.list.forEach(d => d.sent = false); // Reset sent flag

      // It's good practice to remove old listeners if re-calling this function
      // DPlayer's event handling might need specific function references to remove.
      // For simplicity, we assume this is called once after danmaku is loaded for a video.

      // Cache handlers to be able to remove them later if necessary
      if (this._timeUpdateHandler) {
        this.dp.off('timeupdate', this._timeUpdateHandler);
      }
      if (this._seekedHandler) {
        this.dp.off('seeked', this._seekedHandler);
      }

      this._timeUpdateHandler = () => {
        const currentTime = this.dp.video.currentTime;
        this.danmaku.list.forEach(danmu => {
          if (!danmu.sent && Math.abs(danmu.time - currentTime) < 0.5) { // Window of 0.5s
            this.dp.danmaku.draw({
              text: danmu.text,
              color: danmu.color || '#ffffff',
              type: danmu.type || 'right', // 'right', 'top', 'bottom'
              time: danmu.time // DPlayer doesn't use time here, but good to pass for consistency
            });
            danmu.sent = true;
          }
        });
      };

      this._seekedHandler = () => {
        const currentTime = this.dp.video.currentTime;
        // When seeking, resend all danmaku that should have appeared before current time
        // and mark future ones as not sent. DPlayer handles overlapping, but clearing and re-adding
        // up to current time might be more robust for this manual loading.
        // However, the original script just reset the 'sent' flag for future danmakus.
        this.dp.danmaku.clear(); // Clear and redraw for seek is often cleaner with manual loading
        this.danmaku.list.forEach(danmu => {
            danmu.sent = danmu.time < currentTime - 0.5; // Mark past as sent
            if (danmu.time >= currentTime -0.5 && danmu.time < currentTime + 5) { // load near view
                 if(!danmu.sent) { // only draw if not marked as sent (e.g. just before current time)
                    this.dp.danmaku.draw({
                        text: danmu.text,
                        color: danmu.color || '#ffffff',
                        type: danmu.type || 'right',
                        time: danmu.time
                    });
                    danmu.sent = true; // mark as sent after drawing
                 }
            }
        });
      };

      this.dp.on('timeupdate', this._timeUpdateHandler);
      this.dp.on('seeked', this._seekedHandler);

      // Initial draw for current time if video is already playing/paused
      if(this.dp.video.currentTime > 0) {
          this._seekedHandler();
      }
    },
  },
  watch: {
    // Watch for changes that require re-initialization or updates
    'videoInfo.url'(newUrl) {
      if (newUrl && !this.dp) { // If URL is set and player isn't up, init
        // this.initPlayer(); // initPlayer is now called in handleVideoSelected after DOM update
      }
    },
  },
  mounted() {
    // Player is initialized on video selection, not necessarily on mount.
    // We could set up the worker here if it's always needed.
    this.updateStatus('请选择视频和弹幕文件');
  },
  beforeUnmount() {
    if (this.dp) {
      this.dp.destroy();
      this.dp = null;
    }
    if (this.danmakuParseWorker) {
      this.danmakuParseWorker.terminate();
      this.danmakuParseWorker = null;
    }
    // Clean up event listeners from dp if they were stored and attached directly
    if (this.dp && this._timeUpdateHandler) {
        this.dp.off('timeupdate', this._timeUpdateHandler);
    }
    if (this.dp && this._seekedHandler) {
        this.dp.off('seeked', this._seekedHandler);
    }
  }
}
</script>

<style>
/* Global styles from original style.css that apply to the overall layout */
body {
    font-family: "HarmonyOS Sans", "PingFang SC", "Helvetica Neue", Arial, sans-serif;
    background: #f6f7fb;
    margin: 0;
    padding: 0;
}

.main-layout {
    display: flex;
    flex-direction: row;
    gap: 32px;
    max-width: 90%;
    margin: 48px auto 48px auto;
    padding: 0 16px;
    justify-content: center;
    align-items: flex-start;
}

@media (max-width: 900px) {
    .main-layout {
        flex-direction: column;
        gap: 20px;
        margin: 10px;
        align-items: center;
        justify-content: flex-start;
    }
}

/* Ensure components take up appropriate space in mobile view */
@media (max-width: 900px) {
    .player-card-vue, /* Using new class names for Vue components */
    .controls-card-vue {
        width: 100%;
        max-width: 100%;
        margin-left: 0;
        margin-right: 0;
    }
}
</style>
