<template>
  <div id="app">
    <nav>
      <input type="text" placeholder="Entrez l'URL de la vidéo" v-model="url" />
      <button @click="load">Charger</button>
    </nav>
    <div id="youtube-wrapper">
      <youtube
        :video-id="id"
        ref="youtube"
        :playerVars="playerVars"
        @ready="getDuration"
        @playing="onPlaying"
      />
    </div>
    <div class="controls">
      <button @click="playVideo">play</button>
      <button @click="pauseVideo">pause</button>
      <div id="bar" @click="switchTime">
        <div
          id="circle"
          :style="{ left: `calc(${this.advancement}% - 9px)` }"
        ></div>
        <div id="line"></div>
      </div>
    </div>
  </div>
</template>

<script>
import io from "socket.io-client";
let interval = null;

export default {
  name: "App",
  data() {
    return {
      url: null,
      id: "BBJa32lCaaY",
      playerVars: {
        autoplay: 1,
        controls: 0,
        modestbranding: 1
      },
      currentTime: 0,
      totalTime: 1,
      socket: io(process.env.VUE_APP_API_ADRESS)
    };
  },
  methods: {
    load() {
      const id = this.url.split("v=")[1].split("&")[0];
      this.socket.emit("changeID", id);
    },
    getDuration() {
      this.player.getDuration().then(time => {
        this.totalTime = time;
      });
    },
    playVideo() {
      this.socket.emit("playVideo");
    },
    onPlaying() {
      this.getDuration();
      interval = setInterval(() => {
        this.player.getCurrentTime().then(time => {
          this.currentTime = time;
          this.socket.emit("refreshTimer", time);
        });
        this.getDuration();
      }, 50);
    },
    pauseVideo() {
      this.socket.emit("pauseVideo");
    },
    switchTime(event) {
      const bar = document.querySelector("#bar");
      const barPosX = bar.offsetLeft;
      const barWidth = bar.offsetWidth;
      let relativeClickPos = ((event.clientX - barPosX) / barWidth) * 100;
      if (relativeClickPos < 0) {
        relativeClickPos = 0;
      }
      const secondsTimer = (relativeClickPos * this.totalTime) / 100;
      this.socket.emit("seekTo", secondsTimer);
    }
  },
  computed: {
    player() {
      return this.$refs.youtube.player;
    },
    advancement() {
      return (this.currentTime / this.totalTime) * 100;
    }
  },
  mounted() {
    this.socket.on("playVideo", () => {
      this.player.playVideo();
    });
    this.socket.on("pauseVideo", () => {
      this.player.pauseVideo();
      clearInterval(interval);
    });
    this.socket.on("seekTo", data => {
      this.player.seekTo(data, true);
      this.playVideo();
    });
    this.socket.on("initialize", data => {
      this.id = data.ID;
      this.currentTime = data.timer;
    });
    this.socket.on("changeID", data => {
      this.id = data;
      this.getDuration();
    });
  }
};
</script>

<style lang="scss">
body,
html {
  margin: 0;
  min-height: 100vh;
  height: 100%;
  text-align: center;
}
nav {
  background-color: #efefef;
  height: 50px;
  display: flex;
  gap: 30px;
  align-items: center;
  justify-content: center;
  margin-bottom: 50px;
  button,
  input {
    height: 30px;
  }
}

button {
  color: white;
  background-color: #2f89fc;
  border: none;
  padding: 5px 15px;
  border-radius: 5px;
  font-weight: bold;
}
.controls {
  display: flex;
  width: 640px;
  margin: auto;
  justify-content: stretch;
  align-items: center;
  gap: 10px;
  height: 30px;
  button {
    height: 100%;
  }
}
#youtube-wrapper {
  pointer-events: none;
}
#bar {
  flex-grow: 1;
  position: relative;
  cursor: pointer;
  height: 30px;
}
#line {
  height: 0;
  border: 1px solid #2f89fc;
  width: 100%;
  position: absolute;
  top: 15px;
}
#circle {
  cursor: pointer;
  height: 18px;
  width: 18px;
  top: 6px;
  position: absolute;
  background-color: #2f89fc;
  border-radius: 9px;
}
</style>
