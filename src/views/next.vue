<template>
  <div class="next-tracks">
    <h1>{{ $t('next.nowPlaying') }}</h1>
    <TrackList
      :tracks="[currentTrack]"
      type="playlist"
      dbclick-track-func="none"
    />
    <h1 v-show="waitingTracks.length > 0"
      >插队播放
      <button @click="player.clearPlayNextList()">清除队列</button>
    </h1>
    <TrackList
      v-show="waitingTracks.length"
      :tracks="waitingTracks"
      type="playlist"
      :highlight-playing-track="false"
      dbclick-track-func="playTrackOnListByID"
      item-key="id+index"
      :extra-context-menu-item="['removeTrackFromQueue']"
    />
    <h1>{{ $t('next.nextUp') }}</h1>
    <TrackList
      :tracks="subPlaylist"
      type="playlist"
      :highlight-playing-track="false"
      dbclick-track-func="playTrackOnListByID"
    />
  </div>
</template>

<script>
import { mapState, mapActions } from 'vuex';
import { getTrackDetail } from '@/api/track';
import TrackList from '@/components/TrackList.vue';

// 播放列表展示歌曲数量 (包括当前播放歌曲, 但不包括插队歌曲)
export const LIST_LEN = 100;

export default {
  name: 'Next',
  components: {
    TrackList,
  },
  data() {
    return {
      playlist: [],
      waitingTracks: [],
    };
  },
  computed: {
    ...mapState(['player']),
    currentTrack() {
      return this.player.currentTrack;
    },
    playlistSource() {
      return this.player.playlistSource;
    },
    subPlaylist() {
      const begin = this.player.current;
      const end = Math.min(begin + 100, this.playlist.length);
      console.log('subPlaylist', begin, end);
      const tracks = [];
      for (let i = begin + 1; i < end; i++) {
        if (!this.playlist[i]) {
          break;
        }
        tracks.push(this.playlist[i]);
      }
      return tracks;
    },
  },
  watch: {
    playlistSource: {
      handler: 'reloadPlaylist',
      immediate: true,
    },
  },
  activated() {
    this.$parent.$refs.scrollbar.restorePosition();
  },
  methods: {
    ...mapActions(['playTrackOnListByID']),
    reloadPlaylist() {
      this.playlist = [];
      const trackIds = this.player.list;
      if (trackIds.length === 0) {
        return;
      }
      // 分批异步请求整个列表的歌曲详情, 每次请求 100 个歌曲详情.
      // 每个请求对应一个 Promise, 调用 Promise.all 创建 Promise
      // 表示异步等待所有的请求完成, 最后组装成完整的列表.
      const total = trackIds.length;
      const splitSize = 100;
      const allPromises = [];
      for (let i = 0; i < total; i += splitSize) {
        const ids = trackIds.slice(i, i + splitSize);
        console.log('fetching tracks begin:', i);
        const promise = getTrackDetail(ids.join(',')).then(data => {
          console.log('fetching done bgein:', i);
          return data.songs;
        });
        allPromises.push(promise);
      }

      Promise.all(allPromises).then(subTracks => {
        const tracks = subTracks.flat();
        this.playlist.push(...tracks);
      });
    },
  },
};
</script>

<style lang="scss" scoped>
h1 {
  margin-top: 36px;
  margin-bottom: 18px;
  cursor: default;
  color: var(--color-text);
  display: flex;
  justify-content: space-between;
  button {
    color: var(--color-text);
    border-radius: 8px;
    padding: 0 14px;
    display: flex;
    justify-content: center;
    align-items: center;
    transition: 0.2s;
    opacity: 0.68;
    font-weight: 500;
    &:hover {
      opacity: 1;
      background: var(--color-secondary-bg);
    }
    &:active {
      opacity: 1;
      transform: scale(0.92);
    }
  }
}
</style>
