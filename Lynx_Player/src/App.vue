<template>
  <div class="container">
    <div class="player">
      <div class="visualizer-controls">
        <button @click="toggleVisualizerType">
          <span class="mdi mdi-autorenew"></span>
        </button>
      </div>

      <canvas id="visualizer"></canvas>
      <div class="control_list">
        <!-- 播放进度条 -->
        <input
          type="range"
          min="0"
          max="100"
          step="0.1"
          v-model="progress"
          @input="seekTrack"
          @change="seekTrack"
          class="progress-bar"
        />

        <div class="controls">
          <!-- 上一首按钮 -->
          <button @click="prevTrack">
            <span class="mdi mdi-skip-previous"></span>
          </button>
          <!-- 播放/暂停按钮 -->
          <button @click="togglePlay">
            <span :class="isPlaying ? 'mdi mdi-pause' : 'mdi mdi-play'"></span>
          </button>
          <!-- 下一首按钮 -->
          <button @click="nextTrack">
            <span class="mdi mdi-skip-next"></span>
          </button>
          <!-- 循环模式按钮 -->
          <button @click="toggleLoopMode">
            <span v-if="currentLoopMode === 0" class="mdi mdi-repeat"></span>
            <span
              v-else-if="currentLoopMode === 1"
              class="mdi mdi-shuffle"
            ></span>
            <span v-else class="mdi mdi-repeat-variant"></span>
          </button>
          <!-- 选择文件按钮 -->
          <label class="custom-file-upload">
            <input
              type="file"
              accept="audio/*"
              multiple
              @change="handleFileUpload"
            />
            <span class="mdi mdi-folder-music"></span>
          </label>
        </div>

        <div class="playlist-title">
          <span class="mdi mdi-playlist-music"></span>
          <h3>播放列表</h3>
        </div>

        <div class="playlist" @scroll="disableAutoScroll">
          <ul>
            <li
              v-for="(track, index) in tracks"
              :key="index"
              :class="{ playing: index === currentTrackIndex }"
              @click="playSpecificTrack(index)"
            >
              <div class="track-name">{{ track.name }}</div>
              <span
                @click.stop="removeTrack(index)"
                class="mdi mdi-trash-can-outline"
              ></span>
            </li>
          </ul>
        </div>
      </div>
    </div>
  </div>
  <!-- 音频播放元素 -->
  <audio
    ref="audioElement"
    @timeupdate="updateProgressDisplay"
    @ended="onTrackEnd"
    @loadedmetadata="initializeAudio"
  ></audio>
  <footer class="footer">
    <!-- <p>&copy; 2024 版权所有 | LynxHawk's Player</p> -->
    <p>&copy; LynxHawk's Player</p>
  </footer>
</template>

<script lang="ts">
import { defineComponent, ref, reactive, onMounted } from "vue";

const DB_NAME = "MusicPlayerDB";
const STORE_NAME = "Tracks";
const DB_VERSION = 1;

let db: IDBDatabase | null = null;

//打开数据库
const openDB = (): Promise<IDBDatabase> => {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(DB_NAME, DB_VERSION);

    request.onupgradeneeded = (event) => {
      db = (event.target as IDBOpenDBRequest).result;
      if (!db.objectStoreNames.contains(STORE_NAME)) {
        db.createObjectStore(STORE_NAME, {
          keyPath: "id",
          autoIncrement: true,
        });
      }
    };

    request.onsuccess = () => {
      db = request.result;
      resolve(db);
    };

    request.onerror = () => {
      reject(request.error);
    };
  });
};

//添加音乐到数据库
const addTrackToDB = (track: { name: string; file: File }) => {
  if (!db) return;
  const transaction = db.transaction(STORE_NAME, "readwrite");
  const store = transaction.objectStore(STORE_NAME);
  store.add(track);
};

//从数据库获取音乐
const getTracksFromDB = (): Promise<
  { id: number; name: string; file: File }[]
> => {
  return new Promise((resolve, reject) => {
    if (!db) {
      resolve([]);
      return;
    }

    const transaction = db.transaction(STORE_NAME, "readonly");
    const store = transaction.objectStore(STORE_NAME);
    const request = store.getAll();

    request.onsuccess = () => {
      resolve(request.result as { id: number; name: string; file: File }[]);
    };

    request.onerror = () => {
      reject(request.error);
    };
  });
};

//从数据库删除音乐
const deleteTrackFromDB = (id: number) => {
  if (!db) return;
  const transaction = db.transaction(STORE_NAME, "readwrite");
  const store = transaction.objectStore(STORE_NAME);
  store.delete(id);
};

export default defineComponent({
  name: "App",
  setup() {
    //音频播放和分析相关变量
    const audioElement = ref<HTMLAudioElement | null>(null); //引用 <audio> 元素，用于控制音频播放和管理音频事件（如播放、暂停、结束等）。
    const audioContext = ref<AudioContext | null>(null); //Web Audio API 的核心对象，用于创建和控制音频处理管道。
    const analyser = ref<AnalyserNode | null>(null); //Web Audio API 的音频分析节点，用于生成音频数据（如频率和时间域数据），主要用于音频可视化。
    const audioSource = ref<MediaElementAudioSourceNode | null>(null); //Web Audio API 中的音频源节点，将 <audio> 元素连接到音频处理管道。
    const dataArray = ref<Uint8Array | null>(null); //用于存储音频频率数据或波形数据，由 AnalyserNode 生成，用于可视化处理。

    //播放状态相关变量
    const isPlaying = ref(false); //当前音频是否正在播放的状态标志。
    const currentTrackIndex = ref(0); //当前正在播放的音轨索引，用于从播放列表中选择音轨。

    //播放列表相关变量
    const tracks = reactive<{ id: number; name: string; file: File }[]>([]); //播放列表，存储音轨信息的数组，包括 ID、名称和文件对象。
    const loopModes = ["顺序循环", "随机播放", "单曲循环"];
    const currentLoopMode = ref(0);

    //进度条相关变量
    const progress = ref(0);
    let trackDuration = 0; //当前音轨的总时长（秒），用于计算进度和调整播放位置。

    const visualizerType = ref<"circle" | "tree">("circle"); // 当前可视化类型 circle or tree

    let autoScrollEnabled = true; // 标志位：控制是否自动滚动

    //自动滚动到当前音轨
    const scrollToCurrentTrack = () => {
      if (!autoScrollEnabled) return; // 如果用户手动滚动，不自动滚动

      const playlist = document.querySelector(".playlist");
      const currentTrack = document.querySelector(".playlist li.playing");

      if (playlist && currentTrack) {
        currentTrack.scrollIntoView({
          behavior: "smooth", // 平滑滚动
          //block: "nearest", // 只滚动到当前元素可见即可
          block: "center", // 将元素滚动到列表的中间
          inline: "nearest", // 如果水平方向不需要滚动，就忽略
        });
      }
    };

    //禁用自动滚动
    const disableAutoScroll = () => {
      autoScrollEnabled = false;

      // 用户停止滚动 3 秒后恢复自动滚动功能
      clearTimeout((window as any).scrollTimeout);
      (window as any).scrollTimeout = setTimeout(() => {
        autoScrollEnabled = true;
      }, 1000);
    };

    //切换可视化类型
    const toggleVisualizerType = () => {
      visualizerType.value =
        visualizerType.value === "circle" ? "tree" : "circle";
    };

    //初始化音频上下文和音频源
    const initializeAudio = () => {
      if (!audioContext.value) {
        audioContext.value = new (window.AudioContext ||
          window.AudioContext)();
        console.log("AudioContext 初始化");
      }

      if (!audioSource.value && audioElement.value) {
        audioSource.value = audioContext.value.createMediaElementSource(
          audioElement.value
        );

        if (!analyser.value) {
          analyser.value = audioContext.value.createAnalyser();
          analyser.value.fftSize = 512;
        }

        audioSource.value.connect(analyser.value);
        analyser.value.connect(audioContext.value.destination);
      }
    };

    //加载音轨
    const loadTrack = async (index: number) => {
      if (audioElement.value) {
        const track = tracks[index];
        audioElement.value.src = URL.createObjectURL(track.file);
        audioElement.value.load(); // 确保音频资源加载
        console.log("音频加载中...");
        await new Promise((resolve) => {
          audioElement.value?.addEventListener("canplaythrough", resolve, {
            once: true,
          });
        });
        console.log("音频加载完成");
      }
    };

    //处理文件上传
    const handleFileUpload = async (event: Event) => {
      const files = (event.target as HTMLInputElement).files;
      if (files) {
        Array.from(files).forEach((file) => {
          const track = { name: file.name, file };
          addTrackToDB(track);
          tracks.push({ id: Date.now(), name: file.name, file });
        });
      }
    };

    //删除音轨
    const removeTrack = (index: number) => {
      const track = tracks[index];
      deleteTrackFromDB(track.id);
      tracks.splice(index, 1);
    };

    //更新播放进度显示
    const updateProgressDisplay = () => {
      if (audioElement.value && !isNaN(audioElement.value.duration)) {
        trackDuration = audioElement.value.duration; // 更新音频总时长
        progress.value = (audioElement.value.currentTime / trackDuration) * 100;
      }
    };

    //调整音轨进度到指定位置
    const seekTrack = () => {
      if (audioElement.value) {
        const seekTime = (progress.value / 100) * trackDuration;
        audioElement.value.currentTime = seekTime;
      }
    };

    // 在播放按钮事件中调用
    const togglePlay = async () => {
      if (!audioElement.value) return;

      try {
        if (!audioContext.value || audioContext.value.state === "suspended") {
          initializeAudio();
          await audioContext.value?.resume();
          console.log("音频上下文已恢复");
        }

        if (audioElement.value.muted) {
          audioElement.value.muted = false; // 取消静音
        }

        if (isPlaying.value) {
          audioElement.value.pause();
          isPlaying.value = false;
          console.log("音频已暂停");
        } else {
          // 等待音频资源加载完成
          if (audioElement.value.readyState < 4) {
            console.log("等待音频资源加载...");
            await new Promise((resolve) => {
              audioElement.value?.addEventListener("canplaythrough", resolve, {
                once: true,
              });
            });
            console.log("音频资源加载完成");
          }

          await audioElement.value.play();
          isPlaying.value = true;
          visualize();
          console.log("音频播放中");
        }
      } catch (error) {
        console.error("播放音频时出错:", error);
      }
    };

    //播放指定音轨
    const playSpecificTrack = (index: number) => {
      currentTrackIndex.value = index;
      loadTrack(index);

      if (audioElement.value) {
        audioElement.value.oncanplaythrough = () => {
          audioElement.value?.play();
          isPlaying.value = true;
          visualize();
          scrollToCurrentTrack(); // 播放时自动滚动到当前歌曲
        };
      }
    };

    //切换循环模式
    const toggleLoopMode = () => {
      currentLoopMode.value = (currentLoopMode.value + 1) % loopModes.length;
    };

    //播放下一首音轨
    const nextTrack = () => {
      const trackCount = tracks.length;
      if (trackCount === 0) return;

      let nextIndex;
      if (currentLoopMode.value === 2) {
        // 随机播放
        nextIndex = Math.floor(Math.random() * trackCount);
      } else {
        nextIndex = (currentTrackIndex.value + 1) % trackCount;
      }

      playSpecificTrack(nextIndex);
    };

    //播放上一首音轨
    const prevTrack = () => {
      const trackCount = tracks.length;
      if (trackCount === 0) return;

      let prevIndex;
      if (currentLoopMode.value === 2) {
        // 随机播放
        prevIndex = Math.floor(Math.random() * trackCount);
      } else {
        prevIndex = (currentTrackIndex.value - 1 + trackCount) % trackCount;
      }

      playSpecificTrack(prevIndex);
    };

    let trackEnding = false; // 防止重复触发标志

    //音轨播放结束事件处理
    const onTrackEnd = () => {
      if (trackEnding) return; // 避免重复调用
      trackEnding = true;

      console.log("当前音频播放结束，准备切换到下一首");

      setTimeout(() => {
        switch (currentLoopMode.value) {
          case 0: // 顺序循环
            nextTrack();
            break;
          case 1: // 随机播放
            const randomIndex = Math.floor(Math.random() * tracks.length);
            playSpecificTrack(randomIndex);
            break;
          case 2: // 单曲循环
            if (audioElement.value) {
              audioElement.value.currentTime = 0;
              audioElement.value.play();
            }
            break;
        }
        trackEnding = false; // 恢复标志位
      }, 100); // 加入小延时，确保状态正确
    };

    //可视化处理
    const visualize = () => {
      const canvas = document.getElementById("visualizer") as HTMLCanvasElement;
      const ctx = canvas.getContext("2d");

      if (!ctx || !analyser.value) return;

      canvas.width = 600;
      canvas.height = 600;

      // 创建默认空数据数组
      const bufferLength = analyser.value
        ? analyser.value.frequencyBinCount
        : 256; // 默认 256 数据
      dataArray.value = new Uint8Array(bufferLength);

      const draw = () => {
        ctx.clearRect(0, 0, canvas.width, canvas.height);

        if (analyser.value) {
          // 如果有音频输入，获取真实数据
          if (dataArray.value) {
            analyser.value.getByteFrequencyData(dataArray.value);
          }
        } else {
          // 如果没有音频输入，填充最小值
          dataArray.value = new Uint8Array(bufferLength).fill(5);
        }

        if (visualizerType.value === "circle") {
          drawCircleVisualizer(ctx, bufferLength);
        } else if (visualizerType.value === "tree") {
          drawTreeVisualizer(ctx, bufferLength);
        }

        requestAnimationFrame(draw);
      };

      const drawCircleVisualizer = (
        ctx: CanvasRenderingContext2D,
        bufferLength: number
      ) => {
        const centerX = canvas.width / 2;
        const centerY = canvas.height / 2;
        const radius = canvas.width / 3;

        // 每个频谱条的角度 取了部分进行延伸
        const totalAngle = Math.PI * 2;
        const segmentAngle = ((totalAngle / bufferLength) * 4) / 3;

        for (let i = 0; i < 192; i++) {
          const value = dataArray.value ? Math.max(dataArray.value[i], 5) : 5;
          const barLength = value / 2;

          const angle = i * segmentAngle;
          const x1 = centerX + Math.cos(angle) * radius;
          const y1 = centerY + Math.sin(angle) * radius;
          const x2 = centerX + Math.cos(angle) * (radius + barLength);
          const y2 = centerY + Math.sin(angle) * (radius + barLength);

          //ctx.strokeStyle = `hsl(${i * 2}, 100%, 50%)`; // 彩色
          // 创建两种颜色的渐变
          const gradient = ctx.createLinearGradient(x1, y1, x2, y2);
          gradient.addColorStop(0, "#cc53ae"); // 第一种颜色
          gradient.addColorStop(1, "#22de9c"); // 第二种颜色

          ctx.strokeStyle = gradient;
          ctx.lineWidth = 3;
          ctx.beginPath();
          ctx.moveTo(x1, y1);
          ctx.lineTo(x2, y2);
          ctx.stroke();
        }
      };

      // 绘制树状可视化
      const drawTreeVisualizer = (
        ctx: CanvasRenderingContext2D,
        bufferLength: number
      ) => {
        const totalBars = bufferLength / 2; // 减少柱子的数量（可选）
        const barWidth = (canvas.width / totalBars) * 0.6; // 增加柱子的宽度
        const barSpacing = (canvas.width / totalBars) * 0.4; // 间距为总宽度的40%

        for (let i = 0; i < totalBars; i++) {
          const barHeight = dataArray.value
            ? (dataArray.value[i] / 255) * canvas.height * 0.8
            : 0;

          // 使用 HSL 生成颜色，色相随索引平滑变化，降低亮度
          const hue = (i / totalBars) * 360; // 色相均匀分布 (0-360)
          const saturation = 100; // 饱和度设为 80%
          const lightness = 40; // 亮度设为 40%，避免过于亮眼

          ctx.fillStyle = `hsl(${hue}, ${saturation}%, ${lightness}%)`;

          // 绘制柱状条，增加间距
          ctx.fillRect(
            i * (barWidth + barSpacing), // x 位置
            canvas.height - barHeight, // y 位置
            barWidth, // 宽度
            barHeight // 高度
          );
        }
      };

      if (isPlaying.value) {
        draw();
      }
    };

    //在组件挂载后执行
    onMounted(async () => {
      await openDB();
      const savedTracks = await getTracksFromDB();
      tracks.push(...savedTracks);

      // 如果播放列表中有歌曲，自动加载第一首
      if (tracks.length > 0 && audioElement.value) {
        loadTrack(0);
        console.log("默认加载第一首歌曲");
      }

      // 初始化可视化
      visualize();

      document.body.addEventListener(
        "click",
        () => {
          if (!audioContext.value) {
            initializeAudio(); // 确保只初始化一次
          }
        },
        { once: true } // 确保事件只执行一次
      );

      if (audioElement.value) {
        audioElement.value.addEventListener(
          "timeupdate",
          updateProgressDisplay
        );
        audioElement.value.addEventListener("ended", onTrackEnd);
      }
    });

    return {
      audioElement,
      handleFileUpload,
      togglePlay,
      toggleLoopMode,
      prevTrack,
      nextTrack,
      removeTrack,
      isPlaying,
      tracks,
      currentTrackIndex,
      loopModes,
      currentLoopMode,
      playSpecificTrack,
      updateProgressDisplay,
      seekTrack,
      progress,
      onTrackEnd,
      initializeAudio,
      toggleVisualizerType,
      disableAutoScroll,
    };
  },
});
</script>

<style scoped>
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100vh;
  width: 100vw;
  background: linear-gradient(to bottom, #372963, #000000);
  color: white;
}

.player {
  text-align: center;
  flex-direction: row;
  display: flex;
  justify-content: center;
  margin: 0 auto;
}

.visualizer-controls {
  display: flex;
  justify-content: center; /* 控件居中 */
  width: 85%; /* 确保控件占据整个宽度 */
}

.visualizer-controls button {
  background-color: transparent;
  color: white;
  border: none;
  cursor: pointer;
  font-size: 24px;
  height: 100px;
}

.visualizer-controls button:hover {
  background-color: #b6b6b636;
  color: #22de9c;
}

canvas {
  background-color: transparent;
  display: block;
  width: 600px;
  height: 600px;
  margin-right: 100px;
}

.control_list {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.controls {
  margin-top: 30px;
  width: 30%;
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
}

button span.mdi,
.custom-file-upload span.mdi {
  font-size: 60px; /* 图标大小 */
  vertical-align: middle; /* 垂直居中 */
}

button {
  margin: 0 5px;
  padding: 10px 20px;
  font-size: 16px;
  background-color: transparent;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  width: 100px;
  border: none;
  transition: background 0.3s; /* 平滑过渡 */
}

button:focus {
  outline: none; /* 确保点击聚焦时也没有白边 */
}

.control_list button:hover {
  background-color: #b6b6b636;
  color: #22de9c;
}

.custom-file-upload {
  display: inline-flex; /* 使用 flex 布局 */
  justify-content: center; /* 水平居中 */
  align-items: center; /* 垂直居中 */
  padding: 10px 20px;
  font-size: 16px;
  background-color: transparent;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  margin: 0 5px;
  width: 100px;
  transition: background 0.3s; /* 平滑过渡 */
  height: 76px;
}

.custom-file-upload:hover {
  background-color: #b6b6b636;
  color: #22de9c;
}

.custom-file-upload input {
  display: none;
}

.playlist {
  text-align: left;
  width: 100%;
  max-height: 380px;
  overflow-y: auto;
  transition: all 0.3s ease-in-out;
  box-sizing: border-box;
  /* 始终保留滚动条空间 */
  scrollbar-gutter: stable;
}

.playlist-title {
  display: flex;
  justify-content: flex-start;
  align-items: center;
  width: 100%;
  height: 30px;
  margin-top: 30px;
}
.playlist-title h3 {
  margin-left: 10px;
}

.playlist-title span {
  font-size: 35px;
}

/* 滚动条基础样式 */
.playlist::-webkit-scrollbar {
  width: 6px; /* 滚动条默认宽度 */
  opacity: 0; /* 初始透明 */
  transition: opacity 0.3s ease-in-out; /* 平滑透明度 */
}

/* 滚动条轨道 */
.playlist::-webkit-scrollbar-track {
  background: transparent; /* 轨道背景透明 */
}

/* 滚动条滑块 */
.playlist::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0); /* 初始完全透明 */
  border-radius: 3px; /* 圆角 */
  transition: background 0.3s ease-in-out; /* 平滑滑块颜色 */
}

/* 鼠标悬停在列表区域时 */
.playlist:hover::-webkit-scrollbar {
  opacity: 1; /* 过渡显示滚动条 */
}

.playlist:hover::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.3); /* 滑块半透明 */
}

/* 鼠标悬浮在滚动条滑块上时 */
.playlist::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.5); /* 滑块更明显 */
}

.playlist ul {
  list-style: none;
  padding: 0;
}

.playlist li {
  cursor: pointer;
  padding: 10px 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  position: relative; /* 为 span 绝对定位提供参考 */
}

.playlist li.playing {
  background-color: #b6b6b636;
  color: #22de9c;
}

.playlist li span {
  opacity: 0; /* 初始透明 */
  transition: opacity 0.2s ease-in-out; /* 平滑过渡 */
  font-size: 25px;
  pointer-events: auto; /* 确保按钮可接收鼠标事件 */
}

.playlist li:hover span {
  opacity: 1; /* 鼠标悬停在 li 上时按钮可见 */
}

.playlist li span:hover {
  opacity: 1; /* 鼠标悬停在按钮上时保持显示 */
}

.track-name {
  flex-grow: 1;
  overflow: hidden;
}

.progress-bar {
  width: 100%;
  margin: 20px 0px 10px 0px;
  -webkit-appearance: none;
  background: #444;
  height: 5px;
  border-radius: 5px;
  cursor: pointer;
  transition: background 0.3s; /* 平滑过渡 */
}

.progress-bar::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 15px;
  height: 15px;
  background: #ffffff;
  border-radius: 50%;
  cursor: pointer;
}

.progress-bar:hover::-webkit-slider-thumb {
  background: #22de9c; /* 鼠标悬停时滑块变色 */
}

.progress-bar:hover {
  background: #666;
}

.footer {
  position: fixed; /* 固定在页面底部 */
  bottom: 0; /* 距离底部 0px */
  left: 0; /* 从左侧开始 */
  width: 100%; /* 宽度占满整个页面 */
  color: #22de9c; /* 字体颜色 */
  text-align: center; /* 文本居中 */
  padding: 10px 0; /* 上下内边距 */
  font-size: 14px; /* 字体大小 */
  box-shadow: 0 -2px 5px rgba(0, 0, 0, 0.3); /* 添加阴影，提升立体感 */
  z-index: 1000; /* 确保不会被其他内容覆盖 */
  background: linear-gradient(to right, #003366, #663399, #ff66cc, #ff9933),
    /* 水平颜色渐变 */
      linear-gradient(to bottom, rgba(19, 9, 6, 1) 0%, rgba(16, 9, 6, 0.5) 100%); /* 垂直方向渐变调整比例 */
  background-blend-mode: multiply; /* 混合模式叠加 */
  height: 50px;
}

.footer p {
  margin: 30px; /* 去掉段落默认外边距 */
  line-height: 1.5; /* 设置行高 */
}

/* 媒体查询：屏幕宽度小于等于 768px 时 */
@media (max-width: 768px) {
  .player {
    flex-direction: column; /* 改为竖直排列 */
    align-items: center;
    justify-content: center;
    max-width: 100%;
    width: 95%; /* 确保宽度100% */
    padding: 0 15px; /* 添加左右内边距 */
    box-sizing: border-box; /* 确保内边距计入宽度 */
  }

  canvas {
    width: 300px; /* 画布宽度100% */
    height: 300px;
    margin-right: 0;
    margin-bottom: 20px;
  }

  .control_list {
    width: 100%; /* 宽度100% */
    max-width: 400px; /* 最大宽度限制 */
    padding: 0; /* 移除内边距 */
  }

  /* 修改图标样式，确保所有图标居中 */
  button span.mdi,
  .custom-file-upload span.mdi {
    font-size: 30px; /* 减小图标大小 */
    display: flex;
    justify-content: center;
    align-items: center;
    width: 100%;
    height: 100%;
  }

  /* 统一基本按钮样式 */
  button {
    width: 50px; /* 统一宽度 */
    height: 50px; /* 统一高度为方形 */
    padding: 0;
    margin: 0 3px;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  /* 控制按钮容器样式 */
  .controls {
    width: 90%; /* 宽度100% */
    justify-content: space-between; /* 两端对齐 */
    padding: 0; /* 移除内边距 */
  }

  .visualizer-controls {
    width: 100%; /* 宽度100% */
    justify-content: flex-start;
    margin-bottom: 10px;
  }

  .visualizer-controls button {
    width: 50px;
    height: 50px;
    font-size: 40px;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  /* 文件上传按钮样式统一 */
  .custom-file-upload {
    width: 50px;
    height: 50px;
    padding: 0;
    margin: 0 3px;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .progress-bar {
    width: 90%;
  }

  .playlist-title {
    width: 100%;
  }

  .playlist {
    width: 100%;
    height: 200px;
  }

  .track-name {
    display: -webkit-box;
    -webkit-line-clamp: 2;
    -webkit-box-orient: vertical;
    line-height: 1.5;
  }

  .footer {
    height: 20px;
  }

  .footer p {
    margin: 0;
  }
}
</style>
