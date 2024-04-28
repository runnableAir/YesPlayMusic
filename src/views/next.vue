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
export const LIST_LEN = 100;

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
    playlistId() {
      return this.player.playlistSource.id;
    },
    subPlaylist() {
      const current = this.player.current;
      const len = this.player.list.length;
      const start = current + 1;
      const end = Math.min(len, current + 100);
      const tracks = this.playlist.slice(start, end);
      const index = tracks.findIndex(e => !e);
      if (index === -1) {
        return {
          tracks,
          requiredLoad: false,
        };
      }
      return {
        tracks: tracks.slice(0, index),
        requiredLoad: true,
      };
    },
  },
  watch: {
    playlistId: {
      handler: 'reloadPlaylist',
      immediate: true,
    },
    'subPlaylist.requiredLoad': function (requiredLoad) {
      console.log('requiredLoad =', requiredLoad);
      if (!this.reloading && requiredLoad) {
        this.loadPlaylist();
      }
    },
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
      const current = this.player.current;
      const loadStart = Math.max(0, current - 50);
      const loadEnd = Math.min(len, current + 150);
      console.log('loadStart =', loadStart, 'loadEnd =', loadEnd);
      const trackIds = this.player.list.slice(loadStart, loadEnd);
      const tracks = await this.fetchTracks(trackIds);
      this.playlist.splice(loadStart, tracks.length, ...tracks);
    },
    fetchTracks(trackIds) {
      if (!trackIds.length) {
        return [];
      }
      return getTrackDetail(trackIds.join(',')).then(data => data.songs);
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
