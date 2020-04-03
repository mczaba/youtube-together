<template>
  <div id="app">
    <nav>
      <div class="videoInput" v-if="socket">
        <input
          type="text"
          placeholder="Entrez l'URL de la vidéo"
          v-model="url"
          @keydown="keydown($event, 'load')"
        />
        <button @click="load">Charger</button>
      </div>
      <div class="roomInputInput">
        <input
          type="text"
          placeholder="Entrez le nom du salon que vous voulez rejoindre"
          v-model="roomInput"
          @keydown="keydown($event, 'connect')"
        />
        <button @click="connect">
          {{ socket ? "Changer de salon" : "Se connecter au salon" }}
        </button>
      </div>
      <label for="name"
        >Pseudo <input type="text" name="name" v-model="name"
      /></label>
    </nav>
    <div class="mainView" v-if="socket">
      <div class="theater">
        <div id="youtube-wrapper">
          <youtube
            :video-id="id"
            ref="youtube"
            :playerVars="playerVars"
            @playing="onPlaying"
            :width="1200"
            :height="790"
          />
        </div>
        <div class="controls">
          <button @click="playVideo"><i class="fas fa-play"></i></button>
          <button @click="pauseVideo"><i class="fas fa-pause"></i></button>
          <div id="bar" @click="switchTime">
            <div
              id="circle"
              :style="{ left: `calc(${this.advancement}% - 9px)` }"
            ></div>
            <div id="line"></div>
          </div>
          <span
            >{{ currentTime | formatTime }} / {{ totalTime | formatTime }}</span
          >
        </div>
      </div>
      <div class="chat">
        <h3 v-if="socket">Vous êtes actuellement dans le salon : {{ room }}</h3>
        <h4>Il y a actuellement {{ people }} personnes dans le salon</h4>
        <div class="chatView">
          <div class="messageList">
            <p v-for="(message, index) in messages" :key="index">
              <b>{{ message.user }}</b> : {{ message.message }}
            </p>
          </div>
        </div>
        <textarea
          v-model="chatMessage"
          @keydown="keydown($event, 'postMessage')"
          placeholder="Tapez votre message ici"
        />
      </div>
    </div>
  </div>
</template>

<script>
import io from "socket.io-client";
let sendTimeInterval = null;

export default {
  name: "App",
  data() {
    return {
      url: null,
      id: null,
      firstplay: false,
      playerVars: {
        autoplay: 1,
        controls: 0,
        modestbranding: 1
      },
      roomInput: null,
      room: null,
      currentTime: 0,
      totalTime: 1,
      socket: null,
      name: "anonyme",
      chatMessage: "",
      messages: [],
      people: 0
    };
  },
  filters: {
    formatTime(value) {
      let string = "";
      const roundedValue = Math.round(value);
      let seconds = roundedValue % 60;
      if (seconds < 10) {
        string = ":0" + seconds;
      } else {
        string = ":" + seconds;
      }
      let minutes = (roundedValue - seconds) / 60;
      let hours = 0;
      if (minutes > 60) {
        const minutesRest = minutes % 60;
        hours = (minutes - minutesRest) / 60;
        minutes = minutesRest;
        if (minutes < 10) {
          string = `${hours}:0${minutes}` + string;
        } else {
          string = `${hours}:${minutes}` + string;
        }
      } else {
        string = minutes + string;
      }
      return string;
    }
  },
  methods: {
    keydown(event, method) {
      if (event.key === "Enter") {
        event.preventDefault();
        this[method]();
      }
    },
    connect() {
      this.room = this.roomInput;
      this.socket = io(process.env.VUE_APP_API_ADRESS, {
        query: "r_var=" + this.roomInput
      });
      this.socket.on("playVideo", () => {
        this.player.playVideo();
      });
      this.socket.on("pauseVideo", () => {
        this.player.pauseVideo();
        clearInterval(sendTimeInterval);
      });
      this.socket.on("seekTo", data => {
        this.player.seekTo(data, true);
        this.playVideo();
      });
      this.socket.on("initialize", data => {
        this.firstplay = false;
        this.id = data.ID;
        this.people = data.people;
        this.currentTime = data.timer;
        if (this.currentTime !== 0) {
          this.currentTime += 3;
        }
      });
      this.socket.on("changeID", data => {
        this.id = data;
        this.currentTime = 0;
        this.getDuration();
      });
      this.socket.on("messageSent", data => {
        const messageList = document.querySelector(".messageList");
        this.messages.push(data);
        if (
          messageList.scrollTop + messageList.clientHeight ===
          messageList.scrollHeight
        ) {
          setTimeout(() => {
            messageList.scrollTop = messageList.scrollHeight;
          }, 5);
        }
      });
    },
    load() {
      clearInterval(sendTimeInterval);
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
      if (!this.firstplay) {
        this.firstplay = true;
        this.player.seekTo(this.currentTime, true);
      } else {
        sendTimeInterval = setInterval(() => {
          this.player.getCurrentTime().then(time => {
            this.currentTime = time;
            this.socket.emit("refreshTimer", time);
          });
        }, 1000);
      }
      this.getDuration();
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
    },
    postMessage() {
      if (this.chatMessage) {
        this.socket.emit("messageSent", {
          user: this.name,
          message: this.chatMessage
        });
        this.chatMessage = null;
      }
    }
  },
  computed: {
    player() {
      return this.$refs.youtube.player;
    },
    advancement() {
      return (this.currentTime / this.totalTime) * 100;
    }
  }
};
</script>

<style lang="scss">
$secondary: #efefef;
$button: #2f89fc;
body,
html {
  margin: 0;
  min-height: 100vh;
  height: 100%;
  text-align: center;
  background-color: $secondary;
  font-family: "Nunito", sans-serif;
}
nav {
  background-color: white;
  height: 50px;
  display: flex;
  gap: 30px;
  align-items: center;
  justify-content: center;
  button,
  input {
    height: 30px;
    margin-right: 15px;
  }
}

button {
  color: white;
  background-color: $button;
  border: none;
  padding: 5px 15px;
  border-radius: 5px;
  font-weight: bold;
}
.controls {
  display: flex;
  width: 100%;
  margin: auto;
  justify-content: stretch;
  align-items: center;
  gap: 10px;
  height: 30px;
  button {
    height: 100%;
  }
}
.mainView {
  display: flex;
  width: 95%;
  height: 100%;
  justify-content: space-evenly;
  align-items: center;
  margin: 25px 25px;
  gap: 50px;
}
.theater {
  width: 1200px;
}

.chat {
  height: 824px;
  width: 500px;
  background-color: white;
  border-radius: 10px;
  .chatView {
    height: 590px;
    text-align: left;
    padding: 15px 30px;
    overflow: auto;
    position: relative;
    .messageList {
      overflow: auto;
      max-height: 560px;
      width: calc(100% - 30px);
      position: absolute;
      bottom: 19px;
      p {
        margin: 5px 0 0 0;
      }
    }
  }
  h3 {
    margin-bottom: 5px;
  }
  h4 {
    margin-top: 0;
  }
  textarea {
    width: calc(100% - 60px);
    height: 80px;
    padding: 10px 10px;
    border: none;
    border-radius: 10px;
    background-color: $secondary;
    resize: none;
    &::placeholder {
      font-family: "Nunito", sans-serif;
    }
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
  margin: 0 10px;
}
#line {
  height: 0;
  border: 1px solid $button;
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
  background-color: $button;
  border-radius: 9px;
}
</style>
