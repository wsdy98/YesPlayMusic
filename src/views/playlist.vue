<template>
  <div v-show="show">
    <div
      class="playlist-info"
      v-if="specialPlaylistInfo === undefined && !isLikeSongsPage"
    >
      <Cover
        :imageUrl="playlist.coverImgUrl | resizeImage(1024)"
        :showPlayButton="true"
        :alwaysShowShadow="true"
        :clickCoverToPlay="true"
        :fixedSize="288"
        type="playlist"
        :id="playlist.id"
        :coverHover="false"
        :playButtonSize="18"
        @click.right.native="openMenu"
      />
      <div class="info">
        <div class="title" @click.right="openMenu"
          ><span class="lock-icon" v-if="playlist.privacy === 10">
            <svg-icon icon-class="lock" /></span
          >{{ playlist.name }}</div
        >
        <div class="artist">
          Playlist by
          <span
            style="font-weight: 600"
            v-if="
              [
                5277771961,
                5277965913,
                5277969451,
                5277778542,
                5278068783,
              ].includes(playlist.id)
            "
            >Apple Music</span
          >
          <a
            v-else
            :href="`https://music.163.com/#/user/home?id=${playlist.creator.userId}`"
            target="blank"
            >{{ playlist.creator.nickname }}</a
          >
        </div>
        <div class="date-and-count">
          {{ $t("playlist.updatedAt") }}
          {{ playlist.updateTime | formatDate }} · {{ playlist.trackCount }}
          {{ $t("common.songs") }}
        </div>
        <div class="description" @click="showFullDescription = true">
          {{ playlist.description }}
        </div>
        <div class="buttons">
          <ButtonTwoTone @click.native="playPlaylistByID()" :iconClass="`play`">
            {{ $t("common.play") }}
          </ButtonTwoTone>
          <ButtonTwoTone
            v-if="playlist.creator.userId !== data.user.userId"
            :iconClass="playlist.subscribed ? 'heart-solid' : 'heart'"
            :iconButton="true"
            :horizontalPadding="0"
            :color="playlist.subscribed ? 'blue' : 'grey'"
            :textColor="playlist.subscribed ? '#335eea' : ''"
            :backgroundColor="
              playlist.subscribed ? 'var(--color-secondary-bg)' : ''
            "
            @click.native="likePlaylist"
          >
          </ButtonTwoTone>
          <ButtonTwoTone
            iconClass="more"
            :iconButton="true"
            :horizontalPadding="0"
            color="grey"
            @click.native="openMenu"
          >
          </ButtonTwoTone>
        </div>
      </div>
      <div class="search-box" v-if="displaySearchInPlaylist">
        <div class="container" :class="{ active: inputFocus }">
          <svg-icon icon-class="search" />
          <div class="input">
            <input
              :placeholder="inputFocus ? '' : $t('playlist.search')"
              v-model.trim="inputSearchKeyWords"
              v-focus="displaySearchInPlaylist"
              @input="inputDebounce()"
              @focus="inputFocus = true"
              @blur="inputFocus = false"
            />
          </div>
        </div>
      </div>
    </div>
    <div class="special-playlist" v-if="specialPlaylistInfo !== undefined">
      <div
        class="title"
        :class="specialPlaylistInfo.gradient"
        @click.right="openMenu"
      >
        <!-- <img :src="playlist.coverImgUrl | resizeImage" /> -->
        {{ specialPlaylistInfo.name }}
      </div>
      <div class="subtitle"
        >{{ playlist.englishTitle }} · {{ playlist.updateFrequency }}
      </div>

      <div class="buttons">
        <ButtonTwoTone
          class="play-button"
          @click.native="playPlaylistByID()"
          iconClass="play"
          color="grey"
        >
          {{ $t("common.play") }}
        </ButtonTwoTone>
        <ButtonTwoTone
          v-if="playlist.creator.userId !== data.user.userId"
          :iconClass="playlist.subscribed ? 'heart-solid' : 'heart'"
          :iconButton="true"
          :horizontalPadding="0"
          :color="playlist.subscribed ? 'blue' : 'grey'"
          :textColor="playlist.subscribed ? '#335eea' : ''"
          :backgroundColor="
            playlist.subscribed ? 'var(--color-secondary-bg)' : ''
          "
          @click.native="likePlaylist"
        >
        </ButtonTwoTone>
        <ButtonTwoTone
          iconClass="more"
          :iconButton="true"
          :horizontalPadding="0"
          color="grey"
          @click.native="openMenu"
        >
        </ButtonTwoTone>
      </div>
    </div>

    <div class="user-info" v-if="isLikeSongsPage">
      <h1>
        <img class="avatar" :src="data.user.avatarUrl | resizeImage" />{{
          data.user.nickname
        }}{{ $t("library.sLikedSongs") }}
      </h1>
    </div>

    <TrackList
      :tracks="filteredTracks"
      :type="'playlist'"
      :id="playlist.id"
      :extraContextMenuItem="
        isUserOwnPlaylist ? ['removeTrackFromPlaylist'] : []
      "
    />

    <Modal
      :show="showFullDescription"
      :close="() => (showFullDescription = false)"
      :showFooter="false"
      :clickOutsideHide="true"
      title="歌单介绍"
      >{{ playlist.description }}</Modal
    >

    <ContextMenu ref="playlistMenu">
      <div class="item">{{ $t("contextMenu.playNext") }}</div>
      <div class="item" @click="likePlaylist(true)">{{
        playlist.subscribed ? "从音乐库删除" : "保存到音乐库"
      }}</div>
      <div class="item" @click="searchInPlaylist()">歌单内搜索</div>
      <div
        class="item"
        v-if="playlist.creator.userId === data.user.userId"
        @click="editPlaylist"
        >编辑歌单信息</div
      >
      <div
        class="item"
        v-if="playlist.creator.userId === data.user.userId"
        @click="deletePlaylist"
        >删除歌单</div
      >
    </ContextMenu>
  </div>
</template>

<script>
import { mapMutations, mapActions, mapState } from "vuex";
import NProgress from "nprogress";
import {
  getPlaylistDetail,
  subscribePlaylist,
  deletePlaylist,
} from "@/api/playlist";
import { getTrackDetail } from "@/api/track";
import { isAccountLoggedIn } from "@/utils/auth";

import ButtonTwoTone from "@/components/ButtonTwoTone.vue";
import ContextMenu from "@/components/ContextMenu.vue";
import TrackList from "@/components/TrackList.vue";
import Cover from "@/components/Cover.vue";
import Modal from "@/components/Modal.vue";

const specialPlaylist = {
  2829816518: {
    name: "欧美私人订制",
    gradient: "gradient-pink-purple-blue",
  },
  2890490211: {
    name: "助眠鸟鸣声",
    gradient: "gradient-green",
  },
  5089855855: {
    name: "夜的胡思乱想",
    gradient: "gradient-moonstone-blue",
  },
  2888212971: {
    name: "全球百大DJ",
    gradient: "gradient-orange-red",
  },
  2829733864: {
    name: "睡眠伴侣",
    gradient: "gradient-midnight-blue",
  },
  2829844572: {
    name: "洗澡时听的歌",
    gradient: "gradient-yellow",
  },
  2920647537: {
    name: "还是会想你",
    gradient: "gradient-dark-blue-midnight-blue",
  },
  2890501416: {
    name: "助眠白噪声",
    gradient: "gradient-sky-blue",
  },
  5217150082: {
    name: "摇滚唱片行",
    gradient: "gradient-yellow-red",
  },
  2829961453: {
    name: "古风音乐大赏",
    gradient: "gradient-fog",
  },
  4923261701: {
    name: "Trance",
    gradient: "gradient-light-red-light-blue ",
  },
  5212729721: {
    name: "欧美点唱机",
    gradient: "gradient-indigo-pink-yellow",
  },
  3103434282: {
    name: "甜蜜少女心",
    gradient: "gradient-pink",
  },
  2829896389: {
    name: "日系私人订制",
    gradient: "gradient-yellow-pink",
  },
  2829779628: {
    name: "运动随身听",
    gradient: "gradient-orange-red",
  },
  2860654884: {
    name: "独立女声精选",
    gradient: "gradient-sharp-blue",
  },
  898150: {
    name: "浪漫婚礼专用",
    gradient: "gradient-pink",
  },
  2638104052: {
    name: "牛奶泡泡浴",
    gradient: "gradient-fog",
  },
  5317236517: {
    name: "后朋克精选",
    gradient: "gradient-pink-purple-blue",
  },
  2821115454: {
    name: "一周原创发现",
    gradient: "gradient-blue-purple",
  },
  3136952023: {
    name: "私人雷达",
    gradient: "gradient-radar",
  },
};

export default {
  name: "Playlist",
  components: {
    Cover,
    ButtonTwoTone,
    TrackList,
    Modal,
    ContextMenu,
  },
  data() {
    return {
      show: false,
      playlist: {
        id: 0,
        coverImgUrl: "",
        creator: {
          userId: "",
        },
        trackIds: [],
      },
      showFullDescription: false,
      tracks: [],
      loadingMore: false,
      lastLoadedTrackIndex: 9,
      displaySearchInPlaylist: false,
      searchKeyWords: "", // 搜索使用的关键字
      inputSearchKeyWords: "", // 搜索框中正在输入的关键字
      inputFocus: false,
      debounceTimeout: null,
    };
  },
  created() {
    if (this.$route.name === "likedSongs") {
      this.loadData(this.data.likedSongPlaylistID);
    } else {
      this.loadData(this.$route.params.id);
    }
  },
  destroyed() {
    window.removeEventListener("scroll", this.handleScroll, true);
  },
  computed: {
    ...mapState(["player", "data"]),
    isLikeSongsPage() {
      return this.$route.name === "likedSongs";
    },
    specialPlaylistInfo() {
      return specialPlaylist[this.playlist.id];
    },
    isUserOwnPlaylist() {
      return (
        this.playlist.creator.userId === this.data.user.userId &&
        this.playlist.id !== this.data.likedSongPlaylistID
      );
    },
    filteredTracks() {
      return this.tracks.filter(
        (track) =>
          track.name
            .toLowerCase()
            .includes(this.searchKeyWords.toLowerCase()) ||
          track.al.name
            .toLowerCase()
            .includes(this.searchKeyWords.toLowerCase()) ||
          track.ar.find((artist) =>
            artist.name
              .toLowerCase()
              .includes(this.searchKeyWords.toLowerCase())
          )
      );
    },
  },
  methods: {
    ...mapMutations(["appendTrackToPlayerList"]),
    ...mapActions(["playFirstTrackOnList", "playTrackOnListByID", "showToast"]),
    playPlaylistByID(trackID = "first") {
      let trackIDs = this.playlist.trackIds.map((t) => t.id);
      this.$store.state.player.replacePlaylist(
        trackIDs,
        this.playlist.id,
        "playlist",
        trackID
      );
    },
    likePlaylist(toast = false) {
      if (!isAccountLoggedIn()) {
        this.showToast("此操作需要登录网易云账号");
        return;
      }
      subscribePlaylist({
        id: this.playlist.id,
        t: this.playlist.subscribed ? 2 : 1,
      }).then((data) => {
        if (data.code === 200) {
          this.playlist.subscribed = !this.playlist.subscribed;
          if (toast === true)
            this.showToast(
              this.playlist.subscribed ? "已保存到音乐库" : "已从音乐库删除"
            );
        }
        getPlaylistDetail(this.id, true).then((data) => {
          this.playlist = data.playlist;
        });
      });
    },
    loadData(id, next = undefined) {
      this.id = id;
      getPlaylistDetail(this.id, true)
        .then((data) => {
          this.playlist = data.playlist;
          this.tracks = data.playlist.tracks;
          NProgress.done();
          if (next !== undefined) next();
          this.show = true;
          this.lastLoadedTrackIndex = data.playlist.tracks.length - 1;
          if (this.playlist.trackCount > this.tracks.length) {
            window.addEventListener("scroll", this.handleScroll, true);
          }
          return data;
        })
        .then(() => {
          if (this.playlist.trackCount > this.tracks.length) {
            this.loadingMore = true;
            this.loadMore();
          }
        });
    },
    loadMore(loadNum = 50) {
      let trackIDs = this.playlist.trackIds.filter((t, index) => {
        if (
          index > this.lastLoadedTrackIndex &&
          index <= this.lastLoadedTrackIndex + loadNum
        )
          return t;
      });
      trackIDs = trackIDs.map((t) => t.id);
      getTrackDetail(trackIDs.join(",")).then((data) => {
        this.tracks.push(...data.songs);
        this.lastLoadedTrackIndex += trackIDs.length;
        this.loadingMore = false;
      });
    },
    handleScroll(e) {
      let dom = document.querySelector("html");
      let scrollHeight = Math.max(dom.scrollHeight, dom.scrollHeight);
      let scrollTop = e.target.scrollingElement.scrollTop;
      let clientHeight =
        dom.innerHeight || Math.min(dom.clientHeight, dom.clientHeight);
      if (clientHeight + scrollTop + 200 >= scrollHeight) {
        if (
          this.lastLoadedTrackIndex + 1 === this.playlist.trackIds.length ||
          this.loadingMore
        )
          return;
        this.loadingMore = true;
        this.loadMore();
      }
    },
    openMenu(e) {
      this.$refs.playlistMenu.openMenu(e);
    },
    deletePlaylist() {
      if (!isAccountLoggedIn()) {
        this.showToast("此操作需要登录网易云账号");
        return;
      }
      let confirmation = confirm(`确定要删除歌单 ${this.playlist.name}？`);
      if (confirmation === true) {
        deletePlaylist(this.playlist.id).then((data) => {
          if (data.code === 200) {
            alert(`已删除歌单 ${this.playlist.name}`);
            this.$router.go(-1);
          } else {
            alert("发生错误");
          }
        });
      }
    },
    editPlaylist() {
      alert("此功能开发中");
    },
    searchInPlaylist() {
      this.displaySearchInPlaylist = !this.displaySearchInPlaylist;
      if (this.displaySearchInPlaylist == false) {
        this.searchKeyWords = "";
        this.inputSearchKeyWords = "";
      } else {
        this.loadMore(500);
      }
    },
    removeTrack(trackID) {
      if (!isAccountLoggedIn()) {
        this.showToast("此操作需要登录网易云账号");
        return;
      }
      this.tracks = this.tracks.filter((t) => t.id !== trackID);
    },
    inputDebounce() {
      if (this.debounceTimeout) clearTimeout(this.debounceTimeout);
      this.debounceTimeout = setTimeout(() => {
        this.searchKeyWords = this.inputSearchKeyWords;
      }, 600);
    },
  },
  directives: {
    focus: {
      inserted: function (el) {
        el.focus();
      },
    },
  },
};
</script>

<style lang="scss" scoped>
.playlist-info {
  display: flex;
  margin-bottom: 72px;
  position: relative;
  .info {
    display: flex;
    flex-direction: column;
    justify-content: center;
    flex: 1;
    margin-left: 56px;
    .title {
      font-size: 36px;
      font-weight: 700;
      color: var(--color-text);

      .lock-icon {
        opacity: 0.28;
        color: var(--color-text);
        margin-right: 8px;
        .svg-icon {
          height: 26px;
          width: 26px;
        }
      }
    }
    .artist {
      font-size: 18px;
      opacity: 0.88;
      color: var(--color-text);
      margin-top: 24px;
    }
    .date-and-count {
      font-size: 14px;
      opacity: 0.68;
      color: var(--color-text);
      margin-top: 2px;
    }
    .description {
      font-size: 14px;
      opacity: 0.68;
      color: var(--color-text);
      margin-top: 24px;
      display: -webkit-box;
      -webkit-box-orient: vertical;
      -webkit-line-clamp: 3;
      overflow: hidden;
      cursor: pointer;
      &:hover {
        transition: opacity 0.3s;
        opacity: 0.88;
      }
    }
    .buttons {
      margin-top: 32px;
      display: flex;
      button {
        margin-right: 16px;
      }
    }
  }
}

.special-playlist {
  margin-top: 192px;
  margin-bottom: 128px;
  border-radius: 1.25em;
  text-align: center;

  @keyframes letterSpacing4 {
    from {
      letter-spacing: 0px;
    }

    to {
      letter-spacing: 4px;
    }
  }

  @keyframes letterSpacing1 {
    from {
      letter-spacing: 0px;
    }

    to {
      letter-spacing: 1px;
    }
  }

  .title {
    font-size: 84px;
    line-height: 1.05;
    font-weight: 700;
    text-transform: uppercase;

    letter-spacing: 4px;
    animation-duration: 0.8s;
    animation-name: letterSpacing4;
    -webkit-text-fill-color: transparent;
    background-clip: text;
    // background-image: linear-gradient(
    //   225deg,
    //   var(--color-primary),
    //   var(--color-primary)
    // );

    img {
      height: 78px;
      border-radius: 0.125em;
      margin-right: 24px;
    }
  }
  .subtitle {
    font-size: 18px;
    letter-spacing: 1px;
    margin: 28px 0 54px 0;
    animation-duration: 0.8s;
    animation-name: letterSpacing1;
    text-transform: uppercase;
    color: var(--color-text);
  }
  .buttons {
    margin-top: 32px;
    display: flex;
    justify-content: center;
    button {
      margin-right: 16px;
    }
  }
}

.gradient-test {
  background-image: linear-gradient(to left, #92fe9d 0%, #00c9ff 100%);
}

[data-theme="dark"] {
  .gradient-radar {
    background-image: linear-gradient(to left, #92fe9d 0%, #00c9ff 100%);
  }
}

.gradient-radar {
  background-image: linear-gradient(to left, #0ba360 0%, #3cba92 100%);
}

.gradient-blue-purple {
  background-image: linear-gradient(
    45deg,
    #89c4f5 0%,
    #6284ff 42%,
    #ff0000 100%
  );
}

.gradient-sharp-blue {
  background-image: linear-gradient(45deg, #00c6fb 0%, #005bea 100%);
}

.gradient-yellow-pink {
  background-image: linear-gradient(45deg, #f6d365 0%, #fda085 100%);
}

.gradient-pink {
  background-image: linear-gradient(45deg, #ee9ca7 0%, #ffdde1 100%);
}

.gradient-indigo-pink-yellow {
  background-image: linear-gradient(
    43deg,
    #4158d0 0%,
    #c850c0 46%,
    #ffcc70 100%
  );
}

.gradient-light-red-light-blue {
  background-image: linear-gradient(
    225deg,
    hsl(190, 30%, 50%) 0%,
    #081abb 38%,
    #ec3841 58%,
    hsl(13, 99%, 49%) 100%
  );
}

.gradient-fog {
  background: linear-gradient(-180deg, #bcc5ce 0%, #929ead 98%),
    radial-gradient(
      at top left,
      rgba(255, 255, 255, 0.3) 0%,
      rgba(0, 0, 0, 0.3) 100%
    );
  background-blend-mode: screen;
}

.gradient-red {
  background-image: linear-gradient(213deg, #ff0844 0%, #ffb199 100%);
}

.gradient-sky-blue {
  background-image: linear-gradient(147deg, #48c6ef 0%, #6f86d6 100%);
}

.gradient-dark-blue-midnight-blue {
  background-image: linear-gradient(213deg, #09203f 0%, #537895 100%);
}

.gradient-yellow-red {
  background: linear-gradient(147deg, #fec867 0%, #f72c61 100%);
}

.gradient-yellow {
  background: linear-gradient(147deg, #fceb02 0%, #fec401 100%);
}

.gradient-midnight-blue {
  background-image: linear-gradient(-20deg, #2b5876 0%, #4e4376 100%);
}

.gradient-orange-red {
  background-image: linear-gradient(147deg, #ffe53b 0%, #ff2525 74%);
}

.gradient-moonstone-blue {
  background-image: linear-gradient(
    147deg,
    hsl(200, 34%, 8%) 0%,
    hsl(204, 35%, 38%) 50%,
    hsl(200, 34%, 18%) 100%
  );
}

.gradient-pink-purple-blue {
  background-image: linear-gradient(
    to right,
    #ff3cac 0%,
    #784ba0 50%,
    #2b86c5 100%
  ) !important;
}

.gradient-green {
  background-image: linear-gradient(
    90deg,
    #c6f6d5,
    #68d391,
    #38b2ac
  ) !important;
}

.user-info {
  h1 {
    font-size: 42px;
    color: var(--color-text);
    .avatar {
      height: 44px;
      margin-right: 12px;
      vertical-align: -7px;
      border-radius: 50%;
      border: rgba(0, 0, 0, 0.2);
    }
  }
}

.search-box {
  display: flex;
  position: absolute;
  right: 20px;
  bottom: -55px;
  justify-content: flex-end;
  -webkit-app-region: no-drag;

  .container {
    display: flex;
    align-items: center;
    height: 32px;
    background: var(--color-secondary-bg-for-transparent);
    border-radius: 8px;
    width: 200px;
  }

  .svg-icon {
    height: 15px;
    width: 15px;
    color: var(--color-text);
    opacity: 0.28;
    margin: {
      left: 8px;
      right: 4px;
    }
  }

  input {
    font-size: 16px;
    border: none;
    background: transparent;
    width: 96%;
    font-weight: 600;
    margin-top: -1px;
    color: var(--color-text);
  }

  .active {
    background: var(--color-primary-bg-for-transparent);
    input,
    .svg-icon {
      opacity: 1;
      color: var(--color-primary);
    }
  }
}

[data-theme="dark"] {
  .search-box {
    .active {
      input,
      .svg-icon {
        color: var(--color-text);
      }
    }
  }
}
</style>
