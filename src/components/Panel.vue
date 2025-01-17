<template>
  <div
    class="panel"
    :style="{ 'background': backgroundColor }">
    <div class="coverBox">
      <img
        v-if="currentCover !== ''"
        class="cover"
        :src="currentCover"/>
    </div>
    <div
      class="info"
      :style="{ 'color': foregroundColor }">
      <div>
        {{ name }}
      </div>
      <div>
        {{ artist }}
      </div>
    </div>
    <div
      class="control"
      :style="{ 'color': foregroundColor }">
      <img
        class="repeat"
        :style="{ 'color': foregroundColor }"
        :src="fileIconPath(playOrderIcon)"
        @click="togglePlayOrder"
      />
      <img
        class="backward"
        :style="{ 'color': foregroundColor }"
        :src="fileIconPath(stepBackwardIcon)"
        @click="playPrev"
      />
      <img
        class="play"
        :style="{ 'color': foregroundColor }"
        :src="fileIconPath(playIcon)"
        @click="togglePlayStatus"
      />
      <img
        class="forward"
        :style="{ 'color': foregroundColor }"
        :src="fileIconPath(stepForwardIcon)"
        @click="playNext"
      />
      <div class="current-time">
        {{ currentTime }}
      </div>
      /
      <div class="duration">
        {{ duration }}
      </div>
    </div>
    <div class="visual">
      <audio id="audio" ref="player">
        <source :src="currentTrack">
      </audio>
      <div id="audio-visual">
      </div>
    </div>
  </div>
</template>

<script>
 import { mapState } from "vuex";
 import AudioMotionAnalyzer from 'audiomotion-analyzer';

 export default {
   name: 'Panel',
   data() {
     return {
       currentTime: "",
       currentCover: "",
       iconCacheDir: "",
       coverCacheDir: "",
       pathSep: "",
       duration: "",
       name: "",
       artist: "",
       backgroundColor: "",
       foregroundColor: "",
       /* Download icon from https://www.iconfont.cn/collections/detail?spm=a313x.7781069.0.da5a778a4&cid=18739 */
       stepBackwardIcon: "step-backward",
       stepForwardIcon: "step-forward",
       playIcon: "play-circle",
       playOrderIcon: "list",
       iconKey: 1,
       audioMotion: Object,
     }
   },
   computed: mapState([
     "currentTrack",
     "currentTrackIndex",
     "fileInfos"
   ]),
   watch: {
     "fileInfos": function() {
       if (this.playOrderIcon === "random") {
         this.playRandom();
       } else {
         if (this.$refs.player.paused) {
           this.playItem(this.fileInfos[this.currentTrackIndex]);
         }
       }
     },
     
     currentTime: function(newVal) {
       this.$emit('getCurrentTime', newVal);
     }
   },
   props: {
   },
   mounted() {
     window.initPanel = this.initPanel;
     window.forward = this.forward;
     window.backward = this.backward;
     window.playNext = this.playNext;
     window.playPrev = this.playPrev;
     window.playRandom = this.playRandom;
     window.togglePlayStatus = this.togglePlayStatus;
     window.togglePlayOrder = this.togglePlayOrder;
     window.updateCover = this.updateCover;
     window.updateLyric = this.updateLyric;
     window.updateLyricColor = this.updateLyricColor;
     window.setAudioMotion = this.setAudioMotion;

     this.audioMotion =  new AudioMotionAnalyzer(
       document.getElementById('audio-visual'),
       {
         source: document.getElementById('audio')
       }
     )     
     let that = this;

     this.$root.$on("playItem", this.playItem);

     this.$root.$on("updatePanelInfo", this.updatePanelInfo);

     this.$refs.player.addEventListener("ended", this.handlePlayFinish);

     this.$refs.player.addEventListener('timeupdate', () => {
       that.currentTime = that.formatTime(that.$refs.player.currentTime);
       that.duration = that.formatTime(that.$refs.player.duration);
     });
   },
   methods: {
     playItem(item) {
       this.$store.commit("updateCurrentTrack", item.path);

       this.playIcon = "pause-circle";

       this.name = item.name;
       this.artist = item.artist;

       this.$refs.player.load();
       this.$refs.player.play();

       this.currentCover = "";
     },

     updateCover(url) {
       console.log(url);
       var dynamicallyId = new Date();
       var src = url + "?cache=" + dynamicallyId;
       this.currentCover = src;
       this.$store.commit("updateCover", src);
     },

     updateLyricColor(color) {
       this.$store.commit("updateLyricColor", color);
     },

     updateLyric(lyric) {
       lyric = decodeURIComponent(escape(window.atob(lyric)))
       let lines = lyric.split('\n');
       let newLyric = [];
       lines.forEach((line, index) => {
         let newLine = {};
         if (!line) {
           return ;
         }
         let pattern = /\[\S*\]/g;
         let time = line.match(pattern)[0];
         let lineLyric = line.replace(time, '');
         time = time.replace(/\[/, '');
         time = time.replace(/\]/, '');
         newLine.index = index;
         newLine.time = time;
         newLine.content = lineLyric.trim();
         if (newLine.content == '') {
           newLine.content = "~";
         }
         newLine.second = (time.split(":")[0] * 60 + parseFloat(time.split(":")[1])).toFixed(0);
         newLyric.push(newLine);
       })
       this.$store.commit("updateLyric", newLyric);
     },

     updatePanelInfo(name, artist) {
       this.name = name;
       this.artist = artist;
     },

     setAudioMotion(colorList) {
       const options = {
         bgColor: '#011a35', // background color (optional) - defaults to '#111'
         dir: 'h',           // add this property to create a horizontal gradient (optional)
         colorStops: colorList
       }
       this.audioMotion.registerGradient( 'myGradient', options );

       this.audioMotion.setOptions({
         showBgColor: true,
         overlay: true,
         bgAlpha: 0,
         showScaleX: false,
         gradient: 'myGradient',
         mode: 2
       })

       this.audioMotion.setCanvasSize(460, 88)
     },

     togglePlayOrder() {
       if (this.playOrderIcon === "list") {
         this.playOrderIcon = "random";
       } else if (this.playOrderIcon === "random") {
         this.playOrderIcon = "repeat";
       } else if (this.playOrderIcon === "repeat") {
         this.playOrderIcon = "list";
       }
     },

     handlePlayFinish() {
       if (this.playOrderIcon === "list") {
         this.playNext();
       } else if (this.playOrderIcon === "random") {
         this.playRandom();
       } else if (this.playOrderIcon === "repeat") {
         this.playAgain();
       }
     },

     initPanel(playOrderIcon, backgroundColor, foregroundColor, iconCacheDir, coverCacheDir, pathSep, defaultCoverPath) {
       this.playOrderIcon = playOrderIcon;
       this.backgroundColor = backgroundColor;
       this.foregroundColor = foregroundColor;
       this.iconCacheDir = iconCacheDir;
       this.coverCacheDir = coverCacheDir;
       this.pathSep = pathSep;
       this.currentCover = defaultCoverPath;

       this.iconKey = new Date();
     },
     
     fileIconPath(iconFile) {
       return this.iconCacheDir + this.pathSep + iconFile + ".svg" + "?cache=" + this.iconKey;
     },

     getBarColors() {
       return [this.backgroundColor, this.foregroundColor, this.foregroundColor]
     },

     formatTime(seconds) {
       if (isNaN(seconds)) {
         return "00:00"
       } else {
         var minutes = Math.floor(seconds / 60);
         minutes = (minutes >= 10) ? minutes : "0" + minutes;
         seconds = Math.floor(seconds % 60);
         seconds = (seconds >= 10) ? seconds : "0" + seconds;
         return minutes + ":" + seconds;
       }
     },

     forward() {
       this.$refs.player.currentTime += 10;
     },

     backward() {
       this.$refs.player.currentTime -= 10;
     },

     togglePlayStatus() {
       if (this.$refs.player.paused) {
         this.$refs.player.play();
         this.playIcon = "pause-circle";
       } else {
         this.$refs.player.pause();
         this.playIcon = "play-circle";
       }
     },

     playPrev() {
       var currentTrackIndex = this.currentTrackIndex;

       if (currentTrackIndex > 0) {
         currentTrackIndex -= 1;
       } else {
         currentTrackIndex = this.fileInfos.length - 1;
       }

       this.playItem(this.fileInfos[currentTrackIndex]);
     },

     playNext() {
       var currentTrackIndex = this.currentTrackIndex;

       if (currentTrackIndex < this.fileInfos.length - 1) {
         currentTrackIndex += 1;
       } else {
         currentTrackIndex = 0;
       }

       this.playItem(this.fileInfos[currentTrackIndex]);
     },

     playRandom() {
       var min = 0;
       var max = this.fileInfos.length;
       var randomIndex = Math.floor(Math.random() * (max - min + 1)) + min;

       this.playItem(this.fileInfos[randomIndex]);
     },

     playAgain() {
       this.playItem(this.fileInfos[this.currentTrackIndex]);
     }
   }
 }
</script>

<style scoped>
 .panel {
   position: absolute;
   bottom: 0;

   width: 100%;

   box-shadow: 0px -4px 3px rgba(30, 30, 30, 0.2);

   display: flex;
   flex-direction: row;
   align-items: center;
 }

 .coverBox {
   width: 60px;
   margin-left: 30px;
 }
 
 .cover {
   width: 100%;
 }

 .info {
   width: 30%;
   user-select: none;
   padding-left: 30px;

   overflow: hidden;
   white-space: nowrap;
   text-overflow: ellipsis;
 }

 .control {
   display: flex;
   flex-direction: row;
   align-items: center;
   width: 40%;
   justify-content: center;
 }

 .visual {
   width: 30%;
   padding-right: 20px;
 }

 .backward {
   cursor: pointer;
   height: 48px;
 }

 .forward {
   cursor: pointer;
   height: 48px;
 }

 .play {
   margin-left: 10px;
   margin-right: 10px;
   cursor: pointer;
   height: 60px;
 }

 .play-order {
   margin-right: 20px;
   height: 60px;
 }

 .repeat {
   margin-right: 20px;
   height: 24px;
 }
 
 .current-time {
   margin-left: 20px;
   margin-right: 5px;
 }

 .duration {
   margin-left: 5px;
 }
</style>
