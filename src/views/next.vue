<template>
  <div class="next-tracks">
    <h1>{{ $t('next.nowPlaying') }}</h1>
    <TrackList
      :tracks="[currentTrack]"
      type="playlist"
      dbclick-track-func="none"
    />
    <h1 v-show="watingTrackIds.length > 0"
      >插队播放
      <button @click="player.clearPlayNextList()">清除队列</button>
    </h1>
    <TrackList
      v-show="watingTrackIds.length > 0"
      :tracks="waitingTracks"
      type="playlist"
      :highlight-playing-track="false"
      dbclick-track-func="playTrackOnListByID"
      item-key="id+index"
      :extra-context-menu-item="['removeTrackFromQueue']"
    />
    <h1>{{ $t('next.nextUp') }}</h1>
    <TrackList
      :tracks="mainTracks"
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
      trackMap: new Map(), // key: track.id
      mainTracks: [],
      waitingTracks: [],
    };
  },
  computed: {
    ...mapState(['player']),
    currentTrack() {
      return this.player.currentTrack;
    },
    mainTrackIds() {
      const offset = this.player.current + 1;
      return this.player.list.slice(offset, offset + 99);
    },
    watingTrackIds() {
      return this.player.playNextList;
    },
  },
  watch: {
    mainTrackIds: {
      handler: 'asyncUpdateMainTrackList',
      immediate: true,
    },
    watingTrackIds: {
      async handler(ids) {
        const tracks = await this.fetchTracks(ids);
        this.waitingTracks = tracks;
      },
      immediate: true,
    },
  },
  activated() {
    this.$parent.$refs.scrollbar.restorePosition();
  },
  methods: {
    ...mapActions(['playTrackOnListByID']),
    async fetchTracks(trackIDs) {
      if (!trackIDs.length) {
        return [];
      }
      console.log('fetching tracks...');
      const data = await getTrackDetail(trackIDs.join(','));
      console.log('done');
      return data.songs;
    },
    asyncUpdateMainTrackList(trackIDs) {
      this.replaceTracks(trackIDs, newTracks => (this.mainTracks = newTracks));
    },
    async replaceTracks(newTrackIDs, replace) {
      if (!newTrackIDs.length) {
        return;
      }
      const total = newTrackIDs.length;
      // Calculate the index of last available track (that can be got from map)
      const index = newTrackIDs.findIndex(id => !this.trackMap.has(id));
      const end = index == -1 ? total : index;
      let tracks = [];
      if (end === total || end >= 5) {
        // Update tracklist view immediately
        const ids = end === total ? newTrackIDs : newTrackIDs.slice(0, end);
        tracks = ids.map(id => this.trackMap.get(id));
        replace(tracks);
      }
      // fetch remain tracks
      if (end < total) {
        const remains = await this.fetchTracks(newTrackIDs.slice(end));
        tracks.push(...remains);
        replace(tracks);
        remains.forEach(track => this.trackMap.set(track.id, track));
      }
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
