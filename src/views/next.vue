<template>
  <div class="next-tracks">
    <h1>{{ $t('next.nowPlaying') }}</h1>
    <TrackList
      :tracks="[currentTrack]"
      type="playlist"
      dbclick-track-func="none"
    />
    <h1 v-show="playNextList.length > 0"
      >插队播放
      <button @click="player.clearPlayNextList()">清除队列</button>
    </h1>
    <TrackList
      v-show="playNextList.length > 0"
      :tracks="playNextTracks"
      type="playlist"
      :highlight-playing-track="false"
      dbclick-track-func="playTrackOnListByID"
      item-key="id+index"
      :extra-context-menu-item="['removeTrackFromQueue']"
    />
    <h1>{{ $t('next.nextUp') }}</h1>
    <TrackList
      :tracks="filteredTracks"
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

export default {
  name: 'Next',
  components: {
    TrackList,
  },
  data() {
    return {
      tracks: new Map(),
      playNextTracks: [],
      filteredTracks: [],
      filteredTracksChange: 0,
    };
  },
  computed: {
    ...mapState(['player']),
    currentTrack() {
      return this.player.currentTrack;
    },
    current() {
      return this.player.current;
    },
    playerShuffle() {
      return this.player.shuffle;
    },
    playNextList() {
      return this.player.playNextList;
    },
    playlistSource() {
      return this.player.playlistSource.id;
    },
  },
  watch: {
    current() {
      this.filteredTracksChange++;
    },
    playerShuffle() {
      this.filteredTracksChange++;
    },
    playlistSource() {
      this.tracks.clear();
      this.filteredTracksChange++;
    },
    filteredTracksChange(count) {
      if (count > 0) {
        this.loadFilteredTracks(this.current + 1);
      }
    },
    playNextList() {
      this.loadPlayNextTracks();
    },
  },
  activated() {
    this.$parent.$refs.scrollbar.restorePosition();
  },
  created() {
    this.loadFilteredTracks(this.current + 1);
    this.loadPlayNextTracks();
  },
  methods: {
    ...mapActions(['playTrackOnListByID']),
    fetchTracks(ids) {
      const localTracks = this.tracks;
      return getTrackDetail(ids.join(',')).then(data => {
        const newTracks = data.songs;
        for (let track of newTracks) {
          localTracks.set(track.id, track);
        }
      });
    },
    getNextTrackIds(start, length) {
      return this.player.list.slice(start, start + length);
    },
    loadFilteredTracks(start) {
      const queryIds = this.getNextTrackIds(start, 100);
      this.fetchThenUpdate(queryIds, tracks => {
        this.filteredTracks = tracks;
        this.filteredTracksChange = 0;
      });
    },
    loadPlayNextTracks() {
      const queryIds = this.playNextList;
      this.fetchThenUpdate(queryIds, tracks => (this.playNextTracks = tracks));
    },
    fetchThenUpdate(queryIds, handle) {
      const ifNoCache = id => !this.tracks.has(id);
      const update = ids => {
        const tracks = ids.map(id => this.tracks.get(id));
        handle(tracks);
      };
      const prefix = queryIds.findIndex(ifNoCache);
      // 如果前面至少有 5 首歌曲在缓存中，则先更新渲染一次
      if (prefix >= 5) {
        update(queryIds.slice(0, prefix));
      }
      // 如果所有歌曲都在缓存中，则可以直接完成更新，无需加载
      if (prefix === -1) {
        update(queryIds);
        return;
      }
      // 加载歌曲再更新列表
      const unloadIds = queryIds.filter(ifNoCache);
      this.fetchTracks(unloadIds).then(() => update(queryIds));
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
