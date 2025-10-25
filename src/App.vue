<template>
  <div id="app">
    <!-- バンドロゴ -->
    <header class="logo-header">
      <h1>503 bad gateway</h1>
      <button @click="toggleFullscreen" class="fullscreen-btn">
        <span class="material-icons">{{ isFullscreen ? 'fullscreen_exit' : 'fullscreen' }}</span>
      </button>
    </header>

    <!-- メインコンテナ -->
    <div class="main-container">
      <!-- 左カラム: 動画プレビュー -->
      <div class="left-column">
        <div class="video-wrapper">
          <div id="player"></div>
        </div>
      </div>

      <!-- 右カラム: アルバム・楽曲リスト -->
      <div class="right-column">
        <div class="album-list">
          <div v-for="(album, albumIndex) in albums" :key="albumIndex" class="album-section">
            <!-- アルバムヘッダー -->
            <div class="album-header" @click="playAlbum(album)">
              <img :src="album.thumbnail" :alt="album.albumTitle" class="album-thumbnail">
              <div class="album-info">
                <h3 class="album-title">{{ album.albumTitle }}</h3>
                <span v-if="album.isNew" class="new-badge">NEW!　オススメ!</span>
              </div>
            </div>
            <!-- 楽曲リスト -->
            <div class="track-list">
              <div
                v-for="(track, trackIndex) in album.timestamps"
                :key="trackIndex"
                class="track-item"
                :class="{ 'active': isCurrentTrack(album, track) }"
                @click="playTrack(album, track)">
                <div class="track-info">
                  <span class="track-name">{{ track.trackName }}</span>
                  <span v-if="track.description" class="track-description">{{ track.description }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- コントロールバー -->
    <div class="control-bar">
      <button @click="prevTrack" class="control-btn">
        <span class="material-icons">skip_previous</span>
      </button>
      <button @click="togglePlay" class="control-btn">
        <span class="material-icons">{{ isPlaying ? 'pause' : 'play_arrow' }}</span>
      </button>
      <button @click="nextTrack" class="control-btn">
        <span class="material-icons">skip_next</span>
      </button>

      <!-- シークバーセクション -->
      <div class="seek-section">
        <span class="time-display">{{ formatTime(currentTime) }}</span>
        <input
          type="range"
          class="seek-bar"
          :value="currentTime"
          :max="duration"
          @input="onSeekBarInput"
          @change="onSeekBarChange"
        />
        <span class="time-display">{{ formatTime(duration) }}</span>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const albums = ref([])
const player = ref(null)
const isPlaying = ref(false)
const currentAlbum = ref(null)
const currentTrack = ref(null)
const currentTime = ref(0)
const duration = ref(0)
const isSeeking = ref(false)
const isFullscreen = ref(false)
let updateInterval = null

// アルバムデータの読み込み
const loadAlbums = async () => {
  try {
    const response = await fetch('/albums.json')
    albums.value = await response.json()
  } catch (error) {
    console.error('Failed to load albums:', error)
  }
}

// YouTube IFrame APIの読み込み
const loadYouTubeAPI = () => {
  return new Promise((resolve) => {
    if (window.YT && window.YT.Player) {
      resolve()
      return
    }

    const tag = document.createElement('script')
    tag.src = 'https://www.youtube.com/iframe_api'
    const firstScriptTag = document.getElementsByTagName('script')[0]
    firstScriptTag.parentNode.insertBefore(tag, firstScriptTag)

    window.onYouTubeIframeAPIReady = () => {
      resolve()
    }
  })
}

// プレイヤーの初期化
const initPlayer = () => {
  player.value = new window.YT.Player('player', {
    height: '100%',
    width: '100%',
    videoId: albums.value[0]?.videoId || '',
    playerVars: {
      autoplay: 0,
      controls: 1,
      modestbranding: 1,
      rel: 0
    },
    events: {
      onReady: onPlayerReady,
      onStateChange: onPlayerStateChange
    }
  })
}

// プレイヤー準備完了
const onPlayerReady = (event) => {
  updateDuration()
  startTimeUpdate()
}

// プレイヤーの状態変化
const onPlayerStateChange = (event) => {
  isPlaying.value = event.data === window.YT.PlayerState.PLAYING
}

// アルバム再生
const playAlbum = (album) => {
  if (!player.value) return

  currentAlbum.value = album
  currentTrack.value = album.timestamps[0]

  player.value.loadVideoById({
    videoId: album.videoId,
    startSeconds: 0
  })
  isPlaying.value = true
}

// トラック再生
const playTrack = (album, track) => {
  if (!player.value) return

  currentAlbum.value = album
  currentTrack.value = track

  if (player.value.getVideoData().video_id !== album.videoId) {
    player.value.loadVideoById({
      videoId: album.videoId,
      startSeconds: track.time
    })
  } else {
    player.value.seekTo(track.time, true)
    player.value.playVideo()
  }
  isPlaying.value = true
}

// 再生/一時停止トグル
const togglePlay = () => {
  if (!player.value) return

  if (isPlaying.value) {
    player.value.pauseVideo()
  } else {
    player.value.playVideo()
  }
}

// 前のトラック
const prevTrack = () => {
  if (!currentAlbum.value || !currentTrack.value) return

  const currentAlbumIndex = albums.value.findIndex(a => a.videoId === currentAlbum.value.videoId)
  const currentTrackIndex = currentAlbum.value.timestamps.findIndex(t => t.time === currentTrack.value.time)

  if (currentTrackIndex > 0) {
    // 同じアルバムの前のトラック
    playTrack(currentAlbum.value, currentAlbum.value.timestamps[currentTrackIndex - 1])
  } else if (currentAlbumIndex > 0) {
    // 前のアルバムの最後のトラック
    const prevAlbum = albums.value[currentAlbumIndex - 1]
    playTrack(prevAlbum, prevAlbum.timestamps[prevAlbum.timestamps.length - 1])
  }
}

// 次のトラック
const nextTrack = () => {
  if (!currentAlbum.value || !currentTrack.value) return

  const currentAlbumIndex = albums.value.findIndex(a => a.videoId === currentAlbum.value.videoId)
  const currentTrackIndex = currentAlbum.value.timestamps.findIndex(t => t.time === currentTrack.value.time)

  if (currentTrackIndex < currentAlbum.value.timestamps.length - 1) {
    // 同じアルバムの次のトラック
    playTrack(currentAlbum.value, currentAlbum.value.timestamps[currentTrackIndex + 1])
  } else if (currentAlbumIndex < albums.value.length - 1) {
    // 次のアルバムの最初のトラック
    const nextAlbum = albums.value[currentAlbumIndex + 1]
    playTrack(nextAlbum, nextAlbum.timestamps[0])
  }
}

// 現在のトラックかどうか判定
const isCurrentTrack = (album, track) => {
  return currentAlbum.value?.videoId === album.videoId &&
         currentTrack.value?.time === track.time
}

// 動画の長さを更新
const updateDuration = () => {
  if (!player.value || !player.value.getDuration) return
  const dur = player.value.getDuration()
  if (dur && !isNaN(dur)) {
    duration.value = dur
  }
}

// 時間の定期更新を開始
const startTimeUpdate = () => {
  if (updateInterval) {
    clearInterval(updateInterval)
  }
  updateInterval = setInterval(() => {
    if (!player.value || !player.value.getCurrentTime || isSeeking.value) return
    const time = player.value.getCurrentTime()
    if (time !== undefined && !isNaN(time)) {
      currentTime.value = time
    }
    updateDuration()
  }, 100)
}

// シークバーのinputイベント（ドラッグ中）
const onSeekBarInput = (event) => {
  isSeeking.value = true
  currentTime.value = parseFloat(event.target.value)
}

// シークバーのchangeイベント（ドラッグ完了）
const onSeekBarChange = (event) => {
  const seekTime = parseFloat(event.target.value)
  if (player.value && player.value.seekTo) {
    player.value.seekTo(seekTime, true)
  }
  isSeeking.value = false
}

// 時間のフォーマット (秒 -> mm:ss)
const formatTime = (seconds) => {
  if (!seconds || isNaN(seconds)) return '0:00'
  const mins = Math.floor(seconds / 60)
  const secs = Math.floor(seconds % 60)
  return `${mins}:${secs.toString().padStart(2, '0')}`
}

// フルスクリーンのトグル
const toggleFullscreen = () => {
  if (!document.fullscreenElement) {
    document.documentElement.requestFullscreen().catch(err => {
      console.error('フルスクリーンエラー:', err)
    })
  } else {
    document.exitFullscreen()
  }
}

// フルスクリーン状態の監視
const handleFullscreenChange = () => {
  isFullscreen.value = !!document.fullscreenElement
}

// マウント時の処理
onMounted(async () => {
  await loadAlbums()
  await loadYouTubeAPI()
  initPlayer()

  // フルスクリーン状態の監視を開始
  document.addEventListener('fullscreenchange', handleFullscreenChange)
})

// アンマウント時の処理
onUnmounted(() => {
  if (updateInterval) {
    clearInterval(updateInterval)
  }
  if (player.value && player.value.destroy) {
    player.value.destroy()
  }
  // フルスクリーンイベントリスナーを削除
  document.removeEventListener('fullscreenchange', handleFullscreenChange)
})
</script>

<style scoped>
/* スタイルは外部CSSファイルで管理 */
</style>
