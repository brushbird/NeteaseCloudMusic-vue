<template>
  <div>
    <div class="player">
      <div class="player-main">
        <div class="player-main-left" @click="toMusicMsg(musicId)">
          <img v-lazy="imgUrl">
        </div>
        <div class="player-main-main" @click="toMusicMsg(musicId)">
          <div>
            <div class="musicname">{{musicName}}</div>
            <div class="artistsname">
              <span v-for="(item, index) in artists">{{item.name}}<span v-if="index < (artists.length - 1)">/</span></span>
            </div>
          </div>
        </div>
        <div class="player-main-right">
          <span class="icon-list" @click="openBottomSheet"></span>
          <span class="icon-next" @click="next"></span>
          <span class="icon-pause" @click="play(false)" v-if="playStatus"></span>
          <span class="icon-play" @click="play(true)" v-else></span>
        </div>
      </div>
      <mu-linear-progress :value="value1" mode="determinate"></mu-linear-progress>
    </div>
    <mu-bottom-sheet :open="bottomSheet" @close="closeBottomSheet" :sheetClass="{maxHeight: bottomSheet}">
      <mu-list :value="musicId" @itemClick="closeBottomSheet">
        <mu-sub-header>
          播放列表({{musicUrlList.length}})
        </mu-sub-header>
        <mu-list-item v-for="(item, index) of musicUrlList"
          :title="item.name + ' - ' + item.artists[0].name"
          :value="item.id"
          class="demo-list"
          @click="changeMusic(index, item.id)">
          <mu-icon value="ic_close" slot="right"/>
        </mu-list-item>
      </mu-list>
    </mu-bottom-sheet>
    <audio :src="musicUrl" id="audio" autoplay @ended="toNext" @loadedmetadata="getFullTime" @timeupdate="getCurrentTime"></audio>
  </div>
</template>

<script>
  import { mapState } from 'vuex'
  export default {
    data () {
      return {
        ind: '',
        value1: 0,
        bottomSheet: false
      }
    },
    computed: mapState({
      musicUrlList: state => state.musicUrlList,
      musicUrl: state => state.nowMusic.nowMusicUrl,
      musicName: state => state.nowMusic.nowName,
      artists: state => state.nowMusic.nowArtists,
      imgUrl: state => state.nowMusic.nowImgurl,
      musicId: state => state.nowMusic.id,
      playStatus: state => state.isPlaying,
      playModelNum: state => state.playModel
    }),
    methods: {
      toNext () {
        if (this.playModelNum === 0 || this.playModelNum === 1) {
          console.log('列表循环模式播放或随机播放')
          this.next()
        } else if (this.playModelNum === 2) {
          console.log('单曲模式播放')
          const audio = document.querySelector('#audio')
          audio.currentTime = 0
          audio.play()
          return
        }
      },
      getFullTime () {
        const audio = document.querySelector('#audio')
        this.fullTime = audio.duration
        // padStart为了使类似时间1:9，显示为01:09
        this.fullMin = Math.floor(this.fullTime / 60).toString().padStart(2, '0')
        this.fullSec = Math.floor(this.fullTime % 60).toString().padStart(2, '0')
        let {fullTime, fullMin, fullSec} = this
        const obj = {fullTime, fullMin, fullSec}
        this.$store.dispatch('getFull', obj)
      },
      getCurrentTime () {
        const audio = document.querySelector('#audio')
        this.currentTime = audio.currentTime
        this.value1 = (audio.currentTime / this.fullTime) * 100
        this.currentMin = Math.floor(this.currentTime / 60).toString().padStart(2, '0')
        this.currentSec = Math.floor(this.currentTime % 60).toString().padStart(2, '0')
        let {currentTime, currentMin, currentSec, value1} = this
        const obj = { currentTime, currentMin, currentSec, value1 }
        this.$store.dispatch('changeCurrent', obj)
      },
      play (bol) {
        console.log(bol)
        const audio = document.querySelector('#audio')
        if (!this.$store.state.isPlaying) {
          audio.play()
        } else {
          audio.pause()
        }
        this.$store.dispatch('changePlayStatus', bol)
      },
      next () {
        let ind = 0
        // 获得下一首歌曲的id
        let id = 0
        if (this.playModelNum === 0 || this.playModelNum === 2) {
          // 如果是歌曲的最后一首，则ind为0，以便下次取到的是第一首
          ind = this.$store.state.nowMusic.ind === this.$store.state.musicUrlList.length - 1 ? 0 : this.$store.state.nowMusic.ind + 1
          id = this.$store.state.musicUrlList[ind].id
        } else {
          ind = Math.floor(Math.random() * this.$store.state.musicUrlList.length)
          id = this.$store.state.musicUrlList[ind].id
        }
        // 由于获取的歌单，没有歌曲的url，需要先ajax请求url，再发送
        this.$http.get(`http://localhost:3000/music/url?id=${id}`)
          .then((res) => {
            // 下一首歌曲的url
            const url = res.data.data[0].url
            const {name, artists, imgUrl} = this.$store.state.musicUrlList[ind]
            const nextObj = {
              id,
              ind: ind,
              nowMusicUrl: url,
              nowName: name,
              nowArtists: artists,
              nowImgurl: imgUrl
            }
            // this.$router.push({path: `/music/${id}`})
            this.$store.dispatch('changePlayStatus', true)
            this.$store.dispatch('changeMusic', nextObj)
          })
      },
      toMusicMsg (id) {
        this.$store.dispatch('changeControllerStatus', false)
        this.$router.push({path: `/music/${id}`})
      },
      closeBottomSheet () {
        this.bottomSheet = false
      },
      openBottomSheet () {
        this.bottomSheet = true
      },
      changeMusic (ind, id) {
        console.log(ind)
        this.$http.get(`http://localhost:3000/music/url?id=${id}`)
          .then((res) => {
            // 下一首歌曲的url
            const url = res.data.data[0].url
            const {name, artists, imgUrl} = this.musicUrlList[ind]
            const Obj = {
              id,
              ind: ind,
              nowMusicUrl: url,
              nowName: name,
              nowArtists: artists,
              nowImgurl: imgUrl
            }
            // this.$router.push({path: `/music/${id}`})
            this.$store.dispatch('changePlayStatus', true)
            this.$store.dispatch('changeMusic', Obj)
          })
      }
    }
  }
</script>

<style lang="scss">
  @import '../assets/scss/icon.scss';
  .player-main {
    height: 0.8rem;
    border-top: 1px solid #EFF2F7;
    display: flex;
    .player-main-left {
      padding: 0.1rem;
      > img {
        width: 0.6rem;
        height: 0.6rem
      }
    }
    .player-main-main {
      flex: 1;
      display: flex;
      align-items: center;
      .musicname {
        font-weight: 500;
        font-size: 0.2rem;
      }
      .artistsname {
        font-size: 0.15rem;
        color: #99A9BF;
        overflow:hidden;
        text-overflow:ellipsis;
        white-space:nowrap;
      }
    }
    .player-main-right {
      display: flex;
      justify-content: center;
      align-items: center;
      > span {
        margin-right: 0.2rem;
        font-size: 0.4rem;
        color: #df2d2d;
      }
    }
  }
  .demo-list {
    border-bottom: 0.1px solid #999;
  }
  .maxHeight {
    max-height: 5rem;
    overflow-y: scroll;
  }
</style>
