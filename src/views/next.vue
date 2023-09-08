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
      return this.player.playlistSource;
    },
  },
  watch: {
    current(x) {
      console.debug(`[debug][next.vue] watch current(=${x})...`);
      this.loadFilteredTracks(x + 1);
    },
    playerShuffle() {
      console.debug(`[debug][next.vue] watch playerShuffle...`);
      this.loadFilteredTracks(this.current + 1);
    },
    playlistSource() {
      console.debug(`[debug][next.vue] watch playList...`);
      this.tracks.clear();
      this.loadFilteredTracks(this.current + 1);
    },
  },
  activated() {
    this.$parent.$refs.scrollbar.restorePosition();
  },
  created() {
    this.loadFilteredTracks(this.current + 1);
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
        console.debug('[debug][next.vue] fetchTracks: loading done');
      });
    },
    getNextTrackIds(start, length) {
      return this.player.list.slice(start, start + length);
    },
    findTrackByIds(ids) {
      return ids.map(id => this.tracks.get(id)).filter(e => e);
    },
    loadFilteredTracks(start) {
      const queryIds = this.getNextTrackIds(start, 100);
      const callback = tracks => (this.filteredTracks = tracks);
      this.fetchThenUpdate(queryIds, callback);
    },
    fetchThenUpdate(queryIds, onResult) {
      const localTracks = this.tracks;
      const ifNoCache = id => !localTracks.has(id);
      const update = ids => {
        const tracks = this.findTrackByIds(ids);
        onResult(tracks);
      };
      const firstUnloadIndex = queryIds.findIndex(ifNoCache);
      console.debug('[debug][next.vue] first unload index:', firstUnloadIndex);
      const isUpdateFirst = firstUnloadIndex >= 5;
      if (isUpdateFirst) {
        console.debug(
          '[debug][next.vue] update list before fetching new tracks '
        );
        update(queryIds.slice(0, firstUnloadIndex));
      }
      if (firstUnloadIndex === -1) {
        console.debug('[debug][next.vue] no tracks need fetch');
        update(queryIds);
        return;
      }
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
