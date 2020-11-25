<template>
  <div id="app">
    <router-view ref="router" @current-player="currentPlayer" @play-card="playCard" :currentPlayerId="currentPlayerId" :start="start" :socketId="socketId" :room="room" @start-game="startGame" @join-room="joinRoom" @create-room="createRoom" :response="response" :host="host" />
  </div>
</template>

<script>
import io from "socket.io-client";
const socket = io("http://localhost:3000")

export default {
  name: 'App',
  data() {
    return {
      roomId: "",
      start: false,
      host: null,
      currentPlayerId: "",
      response: {
        error: null,
        message: null
      },
      room: {
        players: []
      },
      socketId: ""
    }
  },
  methods: {
    joinRoom(code) {
      if (this.roomId) return;

      socket.emit("join-room", code);
    },
    createRoom() {
      socket.emit("create-room");
    },
    startGame() {
      socket.emit("start-game");
    },
    currentPlayer(id) {
      socket.emit("current-player", id);
    },
    playCard(data) {
      socket.emit("play-card", data)
    }
  },
  mounted() {
    socket.on("connect", () => this.socketId = socket.id);

    socket.on("created-room", id => {
      this.roomId = id;
      this.host = true;

      this.$router.push({ name: "Game", query: { roomCode: this.roomId } })
    })

    socket.on("join-room-response", res => {
      this.response = res;

      if (!this.response.error) {
        this.$router.push({ name: "Game", query: { roomCode: this.room.id } })
      }
    })

    socket.on("start-the-game", () => {
      this.start = true;
    })

    socket.on("room-data", data => {
      this.room = data;
    })

    socket.on("current-player-update", id => {
      this.currentPlayerId = id;
    })

    socket.on("played-card", data => {
      this.$refs.router.otherPlayedCard(data.id, data.card);
    })

    const roomCode = this.$route.query.roomCode;

    if (roomCode) {
      this.$router.push({ name: "Home", query: { roomCode: roomCode } });
      socket.emit("join-room", roomCode);
    }
  }
}
</script>

<style lang="scss">
body {
  width: 100%;
  height: 100vh;
}

#app {
  width: 100%;
  height: 100%;
  background-color: #780E09;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}
</style>