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
      :tracks="subPlaylist.tracks"
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
const LIST_LEN = 100;

export default {
  name: 'Next',
  components: {
    TrackList,
  },
  data() {
    return {
      playlist: [],
      reloading: false,
      waitingTracks: [],
    };
  },
  computed: {
    ...mapState(['player']),
    currentTrack() {
      return this.player.currentTrack;
    },
    subPlaylist() {
      const start = this.displayRange.start;
      const end = this.displayRange.end;
      const tracks = this.playlist.slice(start, end);
      const index = tracks.findIndex(e => !e);
      return {
        tracks: index !== -1 ? tracks.slice(0, index) : tracks,
        requiredLoad: index !== -1,
      };
    },
    displayRange() {
      const current = this.player.current;
      const len = this.player.list.length;
      const start = current + 1;
      const end = Math.min(len, current + LIST_LEN);
      return { start, end };
    },
  },
  watch: {
    'player.list': {
      handler: 'reloadPlaylist',
      immediate: true,
    },
    'subPlaylist.requiredLoad': function (requiredLoad) {
      console.log('requiredLoad =', requiredLoad);
      if (!this.reloading && requiredLoad) {
        this.loadPlaylist();
      }
    },
    'player.playNextList': 'loadWaitingTracks',
  },
  activated() {
    this.$parent.$refs.scrollbar.restorePosition();
  },
  methods: {
    ...mapActions(['playTrackOnListByID']),
    reloadPlaylist() {
      this.reloading = true;
      this.playlist = new Array(this.player.list.length);
      this.loadPlaylist().then(() => (this.reloading = false));
    },
    async loadPlaylist() {
      const len = this.playlist.length;
      // 根据当前列表展示区域计算要加载的歌曲的范围
      const prelaod = 50; // 额外加载的数量
      const start = this.displayRange.start;
      const end = this.displayRange.end;
      const loadStart = Math.max(0, start - prelaod);
      const loadEnd = Math.min(len, end + prelaod);
      console.log('loadStart =', loadStart, 'loadEnd =', loadEnd);
      // 获取在范围[loadStart,loadEnd)中需要加载的歌曲id
      const trackIDs = this.player.list
        .slice(loadStart, loadEnd)
        .filter((_, index) => !this.playlist[loadStart + index]);
      console.log('slice & filtered ', trackIDs.length);
      const fetchedTracks = await this.fetchTracks(trackIDs);
      const tracks = [];
      for (let i = loadStart, j = 0; i < loadEnd; i++) {
        const track = !this.playlist[i] ? fetchedTracks[j++] : this.playlist[i];
        tracks.push(track);
      }
      this.playlist.splice(loadStart, tracks.length, ...tracks);
    },
    loadWaitingTracks() {
      const trackIDs = this.player.playNextList;
      const len = this.waitingTracks.length;
      // 根据当前列表更新的特点进行加载
      // 如果当前列表数量只是变动了 "1", 意味着队列新增或删除了一首歌曲
      // 否则就完整的请求一次列表中的歌曲
      if (Math.abs(trackIDs.length - len) !== 1) {
        this.fetchTracks(trackIDs).then(
          tracks => (this.waitingTracks = tracks)
        );
        return;
      }
      // 找到第一个id不同的歌曲的下标
      // 如果找不到 (= -1), 说明是变动是新增歌曲, 否则说明该下标的歌曲被删除
      const index = this.waitingTracks.findIndex(
        (track, i) => track.id !== trackIDs[i]
      );
      // 删除了歌曲
      if (index !== -1) {
        this.waitingTracks.splice(index, 1);
        return;
      }
      // 新添加的歌曲
      this.fetchTracks(trackIDs.slice(index)).then(tracks =>
        this.waitingTracks.push(tracks[0])
      );
    },
    async fetchTracks(trackIDs) {
      if (!trackIDs.length) {
        return [];
      }
      return getTrackDetail(trackIDs.join(',')).then(data => data.songs);
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
