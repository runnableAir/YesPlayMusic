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

// 播放列表展示歌曲数量 (包括当前播放歌曲, 但不包括插队歌曲)
export const LIST_LEN = 100;

export default {
  name: 'Next',
  components: {
    TrackList,
  },
  data() {
    return {
      tracks: [],
      loadTracksUpdate: 0,
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
    playlistSource() {
      return this.player.playlistSource.id;
    },
    filteredTracks() {
      // 展示列表后99首
      const trackIDs = this.getNextNTrackIds(this.current + 1, LIST_LEN - 1);
      return this.peekTracks(trackIDs);
    },
    playNextList() {
      return this.player.playNextList;
    },
    playNextTracks() {
      return this.peekTracks(this.playNextList);
    },
  },
  watch: {
    current() {
      this.loadTracksUpdate++;
    },
    playerShuffle() {
      this.loadTracksUpdate++;
    },
    playNextList() {
      this.loadPlayNextTracks();
    },
    playlistSource() {
      // 列表源发生了改变，应清除原本缓存的歌曲
      this.tracks = [];
      this.loadTracksUpdate++;
    },
    loadTracksUpdate(count) {
      if (count) {
        this.loadTracks();
        this.loadTracksUpdate = 0;
      }
    },
  },
  activated() {
    this.$parent.$refs.scrollbar.restorePosition();
  },
  created() {
    this.loadTracks();
    this.loadPlayNextTracks();
  },
  methods: {
    ...mapActions(['playTrackOnListByID']),
    loadTracks() {
      // 当前播放下标
      const current = this.current;
      // 预缓存长度
      const cacheLen = 50;
      // 查询150首(列表100首+预缓存50)
      let queryIds = this.getNextNTrackIds(current, LIST_LEN + cacheLen);
      // 检查从头开始均为已加载歌曲的列表前缀长度
      // 如果当该长度小于当前列表长度, 则需要加载其他未加载的歌曲
      const len = this.getLoadLength(queryIds);
      if (len < LIST_LEN && len != queryIds.length) {
        queryIds = queryIds.filter(id => !this.findTrack(id));
        this.fetchTracks(queryIds);
      }
    },
    loadPlayNextTracks() {
      let queryIds = this.playNextList;
      queryIds = queryIds.filter(id => !this.findTrack(id));
      this.fetchTracks(queryIds);
    },
    peekTracks(ids) {
      const len = this.getLoadLength(ids);
      ids = len < ids.length ? ids.slice(0, len) : ids;
      return ids.map(this.findTrack);
    },
    getLoadLength(trackIds) {
      // 返回从该id数组表示的歌曲列表中从头开始均为已加载的最大前缀长度
      const cnt = trackIds.findIndex(id => !this.findTrack(id));
      return cnt === -1 ? trackIds.length : cnt;
    },
    findTrack(id) {
      return this.tracks.find(t => t.id === id);
    },
    getNextNTrackIds(offset, limit) {
      return this.player.list.slice(offset, offset + limit);
    },
    fetchTracks(trackIDs) {
      if (!trackIDs.length) {
        return;
      }
      getTrackDetail(trackIDs.join(',')).then(data => {
        const newTracks = data.songs.filter(x => !this.findTrack(x.id));
        this.tracks.push(...newTracks);
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
