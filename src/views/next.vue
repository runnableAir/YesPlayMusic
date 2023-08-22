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
    };
  },
  computed: {
    ...mapState(['player']),
    currentTrack() {
      return this.player.currentTrack;
    },
    playerShuffle() {
      return this.player.shuffle;
    },
    playNextList() {
      return this.player.playNextList;
    },
  },
  watch: {
    currentTrack() {
      this.loadTracks().then(() => {
        this.loadFilteredTracks();
      });
    },
    playerShuffle() {
      this.loadTracks().then(() => {
        this.loadFilteredTracks();
      });
    },
    playNextList() {
      this.loadTracks().then(() => {
        this.loadPlayNextTracks();
      });
    },
  },
  activated() {
    this.$parent.$refs.scrollbar.restorePosition();
  },
  created() {
    this.loadTracks().then(() => {
      this.loadPlayNextTracks();
      this.loadFilteredTracks();
    });
  },
  methods: {
    ...mapActions(['playTrackOnListByID']),
    loadTracks() {
      // 获取播放列表当前歌曲后100首歌
      let trackIDs = this.player.list.slice(
        this.player.current + 1,
        this.player.current + 100
      );

      // 将playNextList的歌曲加进trackIDs
      trackIDs.push(...this.playNextList);

      // 获取未加载的歌曲
      const unloadedTrackIDs = [];
      const loaded = this.tracks;
      for (let id of trackIDs) {
        if (!loaded.has(id)) {
          unloadedTrackIDs.push(id);
        }
      }

      if (!unloadedTrackIDs.length) {
        return Promise.resolve();
      }
      return getTrackDetail(unloadedTrackIDs.join(',')).then(data => {
        const newTracks = data.songs;
        for (let track of newTracks) {
          loaded.set(track.id, track);
        }
      });
    },
    findTrackByIds(ids) {
      return ids.map(id => this.tracks.get(id) || {});
    },
    loadFilteredTracks() {
      let trackIDs = this.player.list.slice(
        this.player.current + 1,
        this.player.current + 100
      );
      this.filteredTracks = this.findTrackByIds(trackIDs);
    },
    loadPlayNextTracks() {
      this.playNextTracks = this.findTrackByIds(this.playNextList);
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
