<script>
import Card from "../components/Card.vue";
import uniqid from "uniqid";

export default {
  name: "Game",
  components: {
    Card,
  },
  props: {
    host: {
      type: Boolean,
      default: false,
    },
    start: {
      type: Boolean,
      default: false,
    },
    room: {
      type: Object,
      default() {
        return {
          players: [],
          usernames: [],
        };
      },
    },
    socketId: {
      type: String,
      default: "",
    },
    currentPlayerId: {
      type: String,
      default: "",
    },
    winner: {
      type: String,
      default: null,
    },
  },
  data() {
    return {
      topCardTransform: null,
      turn: null,
      top: {
        count: 0,
        id: "",
      },
      left: {
        count: 0,
        id: "",
      },
      right: {
        count: 0,
        id: "",
      },
      pile: [],
      cards: [],
      playDirectionReverse: false,
      pickColor: false,
      wildcardColor: null,
      wildcardTemp: null,
      drawing: false,
      winnerFound: false,
      winnerData: {
        pos: null,
        username: null,
      },
      hasCalledUno: false,
      stack: 0,
      stackCardAmount: 0,
    };
  },
  computed: {
    playableCardsCount() {
      return this.cards.filter((card) => card.playable).length;
    },
    canDraw() {
      if (this.turn === "you" && this.playableCardsCount === 0) {
        return true;
      } else {
        return false;
      }
    },
    canStartGame() {
      return (
        this.room.host === this.socketId &&
        this.room.players.length === 4 &&
        !this.start
      );
    },
    playerCount() {
      return this.room.players.length;
    },
  },
  watch: {
    pile() {
      if (this.turn === "you") {
        this.findPlayable();
      }
    },
    start() {
      if (this.start) {
        this.startGame();
      }
    },
    room() {
      this.setPlayers();
    },
    currentPlayerId(val) {
      this.turn = this.getPosFromId(val);
    },
    turn() {
      if (this.turn === "you") {
        this.findPlayable();
      }
    },
    wildcardColor() {
      this.pickColor = false;
      this.wildcardTemp.card.number = this.wildcardColor;
      this.cardClicked(this.wildcardTemp, true);
    },
    cards(val) {
      if (val.length === 0) {
        this.$emit("has-won");
      } else if (val.length !== 2) {
        this.hasCalledUno = false;
      }
    },
    winner(id) {
      if (id === null) return;

      this.winnerData.pos = this.getPosFromId(id);

      if (this.winnerData.pos === "you") {
        this.winnerData.username = "you";
      } else {
        this.winnerData.username = this.getUsernameFromId(id);
      }

      this.winnerFound = true;
    },
  },
  methods: {
    checkIfPlayerCanStack(amount, stack) {
      let canStack = false;
      this.cards.forEach((card) => {
        if (amount === 2 && card.number === 11) {
          canStack = true;
        } else if (amount === 4 && card.color === "plus4") {
          canStack = true;
        }
      });

      if (canStack) {
        this.stack = stack;
        this.stackCardAmount = amount;
        this.findPlayable();
      } else {
        this.$emit("cant-stack", stack);
        this.nextPlayer(this.getNextPlayerPos());
      }
    },
    copyJoinRoomLink() {
      const link = `${window.location.origin}/game?room=${this.room.id}`;
      window.navigator.clipboard
        .writeText(link)
        .then(() => alert("Copied!"))
        .catch((err) =>
          alert(`Sorry we couldn't copy the link to the clipboard: ${err}`)
        );
    },
    resetGame(goToHome) {
      this.$emit("reset-game");

      if (goToHome) {
        this.$router.push("/");
      }
    },
    getUsernameFromId(id) {
      const index = this.room.players.findIndex((player) => player === id);
      return this.room.usernames[index];
    },
    getPosFromId(id) {
      switch (id) {
        case this.left.id:
          return "left";
        case this.right.id:
          return "right";
        case this.top.id:
          return "top";
        default:
          return "you";
      }
    },
    otherPlayedCard(id, card) {
      const pos = this.getPosFromId(id);
      card.pos = pos;

      if (pos === "you") return;

      const ref = pos + this[pos].count;
      this.$refs[ref][0].clicked({ target: this.$refs[ref][0].$el }, card);
    },
    getNextPlayerPos() {
      return this.playDirectionReverse ? "left" : "right";
    },
    nextPlayer(player) {
      // switch to next player based on play direction
      this.$emit("current-player", this[player].id);
    },
    setPlayers() {
      if (this.room.players.length !== 4) return;
      const binds = ["right", "top", "left"];
      const me = this.room.players.indexOf(this.socketId);
      let i = me + 1;
      let count = 0;

      while (i !== me) {
        if (i > this.room.players.length - 1) {
          i = 0;
        }

        if (!this[binds[count]]) {
          break;
        }

        this[binds[count]].id = this.room.players[i];

        count++;
        i++;
      }
    },
    sortCards() {
      const red = this.cards
        .filter((card) => card.color === "red")
        .sort((a, b) => a.number - b.number);
      const green = this.cards
        .filter((card) => card.color === "green")
        .sort((a, b) => a.number - b.number);
      const yellow = this.cards
        .filter((card) => card.color === "yellow")
        .sort((a, b) => a.number - b.number);
      const blue = this.cards
        .filter((card) => card.color === "blue")
        .sort((a, b) => a.number - b.number);
      const changeColor = this.cards.filter(
        (card) => card.color === "changeColor"
      );
      const plus4 = this.cards.filter((card) => card.color === "plus4");

      this.cards = [
        ...red,
        ...green,
        ...yellow,
        ...blue,
        ...changeColor,
        ...plus4,
      ];
    },
    async giveCards(num, person = "you", anims = true, tellServer = false) {
      for (let i = 0; i < num; i++) {
        this.addCard(person, anims, tellServer);
        if (anims) await this.sleep(500);
      }

      person === "you" ? this.sortCards() : null;
      return;
    },
    sleep(ms) {
      return new Promise((resolve) => setTimeout(resolve, ms));
    },
    generateCard(hidden = false) {
      let colors = ["red", "green", "yellow", "blue"];
      let color = colors[Math.floor(Math.random() * 4)];

      let number = Math.floor(Math.random() * 13) + 1;
      if (number === 10) number = 0;

      // special card chance
      if (Math.random() < 0.06) {
        color = ["plus4", "changeColor"][Math.floor(Math.random() * 2)];
        number = 1;
      }

      return {
        color,
        number,
        id: uniqid.time(),
        hidden,
      };
    },
    addCard(person = "you", anims = true, tellServer) {
      const card = this.generateCard(anims);

      if (person === "you") {
        this.cards.push(card);

        if (tellServer) {
          this.$emit("draw-card", 1);
        }
      } else {
        this[person].count++;
      }

      if (!anims) return;

      if (person === "you") this.drawing = true;

      const observer = new MutationObserver((mutations, me) => {
        let length = this.cards.length;

        if (person !== "you") {
          length = person === "right" ? 1 : this[person].count;
        }

        const element = document.querySelector(
          `.cards.${person} .card:nth-of-type(${length})`
        );

        if (element) {
          const { x, y } = this.$refs.topCard.$el.getBoundingClientRect();
          const { x: desX, y: desY } = element.getBoundingClientRect();

          if (person === "left") {
            this.topCardTransform = `translate(${(x - desX * 0.96) * -1}px, ${
              (y - desY) * -1
            }px) rotateX(-25deg) rotateY(52deg) scale(.66) !important`;
          } else if (person === "right") {
            this.topCardTransform = `translate(${(x - desX * 0.96) * -1}px, ${
              (y - desY) * -1
            }px) rotateX(25deg) rotateY(52deg) scale(.66) !important`;
          } else if (person === "top") {
            this.topCardTransform = `translate(${(x - desX * 0.96) * -1}px, ${
              (y - desY) * -1
            }px) rotateX(-30deg) scale(.65) !important`;
          } else {
            let rotate = element.style.transform.match(/[rotate](.*)/)[0];
            rotate = rotate.slice(7, rotate.length - 1);

            this.topCardTransform = `translate(${(x - desX * 0.96) * -1}px, ${
              (y - desY) * -1
            }px) rotate(${rotate}) rotateY(180deg) !important`;
          }

          this.$refs.topCard.$el.style.zIndex = 1000;

          setTimeout(() => {
            if (person === "you") {
              this.cards.splice(this.cards.length - 1, 1, {
                ...card,
                hidden: false,
              });
            }

            this.topCardTransform = null;
            this.findPlayable();
            this.drawing = false; // allow player to draw another card once animation is played
          }, 450);

          me.disconnect(); // stop observing
          return;
        }
      });

      observer.observe(document, {
        childList: true,
        subtree: true,
      });
    },
    async startGame() {
      this.cards = [];
      this.pile = [];

      await this.giveCards(7, undefined, false);
      await this.giveCards(7, "left", false);
      await this.giveCards(7, "top", false);
      await this.giveCards(7, "right", false);

      if (this.host) {
        this.findPlayable();
        this.$emit("current-player", this.socketId);
      }
    },
    findPlayable() {
      if (this.stack > 0) {
        this.cards.forEach((card) => {
          const temp = { ...card };
          temp.playable = false;

          if (this.stackCardAmount === 2 && temp.number === 11) {
            temp.playable = true;
          } else if (this.stackCardAmount === 4 && temp.color === "plus4") {
            temp.playable = true;
          }

          card = temp;
        });

        return;
      }

      const indexes = [];
      const topCard = this.pile[this.pile.length - 1];

      if (!topCard) {
        this.cards.forEach((card) => (card.playable = true));

        this.cards.forEach((cardOld, i) => {
          const card = { ...cardOld };

          card.playable = true;
          card.id = uniqid.time();

          this.cards.splice(i, 1, card);
        });
        return;
      }

      this.cards.forEach((card, i) => {
        card.playable = false;

        /*
         *   Checks for when topCard is a wildcard - card number relates to a color in spritesheet
         */
        if (topCard.color === "plus4" || topCard.color === "changeColor") {
          if (topCard.number === 2 && card.color === "yellow") {
            return indexes.push(i);
          } else if (topCard.number === 3 && card.color === "red") {
            return indexes.push(i);
          } else if (topCard.number === 4 && card.color === "blue") {
            return indexes.push(i);
          } else if (topCard.number === 5 && card.color === "green") {
            return indexes.push(i);
          }
        }

        if (card.color === "plus4" || card.color === "changeColor") {
          return indexes.push(i);
        } else if (card.color === topCard.color) {
          return indexes.push(i);
        } else if (
          card.number === topCard.number &&
          topCard.color !== "plus4" &&
          topCard.color !== "changeColor"
        ) {
          return indexes.push(i);
        } else {
          return;
        }
      });

      indexes.forEach((i) => {
        const card = { ...this.cards[i] };

        card.playable = true;
        card.id = uniqid.time();

        this.cards.splice(i, 1, card);
      });

      this.sortCards();
    },
    cardClicked(e, skip = false) {
      if (this.stack > 0) {
        this.stack = 0;
        this.stackCardAmount = 0;
      }

      if (!e.card.other) {
        // check if player should be punished for not calling uno
        let punishUno = false;
        if (this.cards.length === 2 && !this.hasCalledUno) {
          punishUno = true;
          this.giveCards(2, "you", false, false);
        }

        if (
          (e.card.color === "plus4" || e.card.color === "changeColor") &&
          !skip
        ) {
          this.pickColor = true;
          this.wildcardTemp = e;
          return;
        }

        this.cards.splice(e.index, 1);
        this.cards = [...this.cards];

        e.card.id = uniqid.time();
        this.pile.push(e.card);

        let playCardRes = {
          id: this.socketId,
          nextPlayer: this.left.id,
          card: {
            color: e.card.color,
            number: e.card.number,
            punishUno,
          },
        };

        if (e.card.number === 12) {
          // skip next player if a skip card is played
          playCardRes.nextPlayer = this.top.id;
          this.nextPlayer("top");
          this.$emit("play-card", playCardRes);
        } else if (e.card.number === 13) {
          // if play direction is reversed play opposite as direction will be reversed
          // once message is sent and recieved from server
          if (this.playDirectionReverse) {
            playCardRes.nextPlayer = this.right.id;
            this.nextPlayer("right");
            this.$emit("play-card", playCardRes);
          } else {
            playCardRes.nextPlayer = this.left.id;
            this.nextPlayer("left");
            this.$emit("play-card", playCardRes);
          }
        } else {
          if (this.playDirectionReverse) {
            this.nextPlayer("left");
            playCardRes.nextPlayer = this.left.id;
            this.$emit("play-card", playCardRes);
          } else {
            this.nextPlayer("right");
            playCardRes.nextPlayer = this.right.id;
            this.$emit("play-card", playCardRes);
          }
        }
      } else {
        const card = {
          ...e.card.other,
          offsetX: e.card.offsetX,
          offsetY: e.card.offsetY,
          rotate: e.card.rotate,
        };

        this[card.pos].count--;
        this[card.pos] = { ...this[card.pos] };

        card.id = uniqid.time();

        this.pile.push(card);

        if (e.card.other.punishUno) {
          setTimeout(() => {
            this.giveCards(2, e.card.other.pos, false, false);
          });
        }
      }

      if (this.pile.length >= 12) {
        this.pile.unshift();
      }
    },
  },
  mounted() {
    this.setPlayers();
  },
};
</script>

<template>
  <div class="game">
    <div v-if="winnerFound" class="winner">
      <div class="card">
        <h1>
          Congratulations to {{ winnerData.username }} on winning the game!
        </h1>

        <button class="btn rounded-btn" @click="resetGame(true)">
          Main Menu
        </button>
      </div>
    </div>

    <div v-if="pickColor" class="color-picker">
      <div class="container">
        <button @click="wildcardColor = 3" class="red"></button>
        <button @click="wildcardColor = 5" class="green"></button>
        <button @click="wildcardColor = 2" class="yellow"></button>
        <button @click="wildcardColor = 4" class="blue"></button>
      </div>
    </div>

    <div class="pile">
      <Card
        v-for="card in pile"
        :key="card.id"
        :color="card.color"
        :number="card.number"
        :offsetX="card.offsetX"
        :offsetY="card.offsetY"
        :pileRotate="card.rotate"
        :pile="true"
      />
    </div>

    <div class="direction" :class="{ reverse: !playDirectionReverse }" />

    <div class="cards other right">
      <Card
        v-for="i in right.count"
        :ref="'right' + i"
        :key="i"
        :color="'plus4'"
        :number="6"
        :style="{ zIndex: i }"
        :length="8"
        :index="i"
        @clicked="cardClicked"
        :other="true"
      />
    </div>
    <div class="cards other left">
      <Card
        v-for="i in left.count"
        :key="i"
        :ref="'left' + i"
        :color="'plus4'"
        :number="6"
        :style="{ zIndex: i }"
        :length="8"
        :index="i"
        @clicked="cardClicked"
        :other="true"
        :left="true"
      />
    </div>
    <div class="cards other top">
      <Card
        v-for="i in top.count"
        :ref="'top' + i"
        :key="i"
        :color="'plus4'"
        :number="6"
        :style="{ zIndex: i }"
        :length="8"
        :index="i"
        @clicked="cardClicked"
        :other="true"
        :top="true"
      />
    </div>

    <div class="hud">
      <div class="cards you">
        <Card
          v-for="(card, i) in cards"
          :key="card.id"
          :color="card.color"
          :number="card.number"
          :style="{ zIndex: i }"
          :length="cards.length"
          :index="i"
          :hidden="card.hidden || false"
          :playable="(card.playable && turn === 'you') || false"
          @clicked="cardClicked"
        />
      </div>
      <div
        class="stack"
        @click="turn === 'you' && !drawing ? addCard('you', true, true) : null"
      >
        <Card :color="'plus4'" :number="6" />
        <Card :color="'plus4'" :number="6" />
        <Card :color="'plus4'" :number="6" />
        <Card :color="'plus4'" :number="6" />
        <Card :color="'plus4'" :number="6" />
        <Card :color="'plus4'" :number="6" />
        <Card :color="'plus4'" :number="6" :class="{ draw: canDraw }" />
        <Card
          :color="'plus4'"
          :number="6"
          ref="topCard"
          :forceTransform="topCardTransform"
          style=""
          :noTransition="!topCardTransform ? true : false"
        />
      </div>
      <div
        v-if="this.start"
        class="player-card you"
        :class="{ playing: this.turn === 'you' }"
      >
        {{ this.getUsernameFromId(this.socketId) }} : {{ this.cards.length }}
      </div>
      <div
        v-if="this.right.id && this.start"
        class="player-card right"
        :class="{ playing: this.turn === 'right' }"
      >
        {{ this.getUsernameFromId(this.right.id) }} : {{ this.right.count }}
      </div>
      <div
        v-if="this.left.id && this.start"
        class="player-card left"
        :class="{ playing: this.turn === 'left' }"
      >
        {{ this.getUsernameFromId(this.left.id) }} : {{ this.left.count }}
      </div>
      <div
        v-if="this.top.id && this.start"
        class="player-card top"
        :class="{ playing: this.turn === 'top' }"
      >
        {{ this.getUsernameFromId(this.top.id) }} : {{ this.top.count }}
      </div>
      <button
        v-if="canStartGame"
        class="start-btn rounded-btn"
        @click="$emit('start-game')"
      >
        Start Game
      </button>
      <button
        v-if="
          this.cards.length === 2 &&
          this.turn === 'you' &&
          !this.hasCalledUno &&
          this.playableCardsCount > 0
        "
        class="uno-btn rounded-btn"
        @click="hasCalledUno = true"
      >
        Call Uno
      </button>
      <div class="top-left-text">
        <p class="room">
          Room Code: {{ room.id }}
          <button
            class="copy"
            style="margin-top: 0px"
            @click="copyJoinRoomLink"
          >
            Copy Link
          </button>
        </p>
        <p class="players">
          Players: {{ playerCount === 0 ? 1 : playerCount }} / 4
        </p>
        <button class="rounded-btn btn" @click="resetGame(true)">
          Leave Game
        </button>
      </div>
    </div>
  </div>
</template>

<style lang="scss">
$mobile: 900px;

.game {
  width: 100%;
  height: 100%;
  background: radial-gradient(
    circle,
    rgb(192, 34, 26) 0%,
    rgb(146, 25, 19) 60%,
    rgb(109, 16, 11) 100%
  );
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

.winner {
  position: absolute;
  top: 0;
  left: 0;
  display: flex;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.3);
  z-index: 1000;

  .card {
    margin: auto;
    background-color: white;
    padding: 35px;
    border-radius: 10px;
    display: flex;
    align-items: center;
    flex-direction: column;

    h1 {
      font-size: 2rem;
      font-weight: bold;
    }

    .btn {
      margin-top: 10px;
    }
  }
}

.rounded-btn {
  padding: 14px 22px;
  background-color: #ff520d;
  color: #fff;
  border: 2px solid white;
  border-radius: 6px;
  font-size: 1.5rem;
  font-weight: bold;
  transition: background-color 0.2s ease;
  outline: none;

  &:hover {
    background-color: #ff8e0d;
  }
}

.hud {
  margin-top: auto;
  z-index: 400;

  @media screen and (max-width: $mobile) {
    .top-left-text {
      font-size: 0.8rem !important;

      .btn {
        transform: scale(0.8);
        transform-origin: top left;
      }
    }

    .stack {
      transform: scale(0.52);
      transform-origin: bottom left;
      margin-left: -30px;
      margin-bottom: -10px;
    }
  }

  .start-btn {
    position: absolute;
    top: 20px;
    right: 20px;
  }

  .uno-btn {
    position: absolute;
    bottom: 60px;
    right: 60px;
    font-size: 2rem;

    @media screen and (max-width: $mobile) {
      transform: scale(0.7);
      bottom: 30px;
      right: 30px;
    }
  }

  .top-left-text {
    position: absolute;
    top: 20px;
    left: 20px;
    color: white;
    font-weight: bold;
    font-size: 1.2rem;

    .btn {
      padding: 5px 10px;
      font-size: 1rem;
      margin-top: 10px;
    }

    .room {
      .copy {
        text-decoration: underline;
        font-weight: bold;
        margin-left: 12px;
        color: #53a944;
        outline: none;
        transition: color 0.2s ease;

        &:hover,
        &:focus {
          color: #50ff31;
        }
      }
    }
  }

  .player-card {
    padding: 12px 22px;
    background-color: white;
    border: 6px solid black;
    border-radius: 8px;
    position: absolute;
    z-index: 100;
    font-weight: bold;

    @media screen and (max-width: $mobile) {
      transform: scale(0.6);
      transform-origin: center;
    }

    &.playing {
      box-shadow: 0px 0px 10px 9px #fcc81c;
    }

    &.right {
      right: 120px;
      top: 41%;

      @media screen and (max-width: $mobile) {
        right: 65px;
        top: 42%;
      }
    }

    &.left {
      left: 90px;
      top: 41%;

      @media screen and (max-width: $mobile) {
        left: 65px;
        top: 42%;
      }
    }

    &.top {
      left: 44.5%;
      top: 160px;

      @media screen and (max-width: $mobile) {
        top: 60px;
      }
    }

    &.you {
      left: 45%;
      bottom: 15px;
      filter: brightness(0.6);

      @media screen and (max-width: $mobile) {
        bottom: 3px;
      }

      &.playing {
        filter: unset;
      }
    }
  }
}

.color-picker {
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.3);
  position: absolute;
  z-index: 1000;
  display: flex;

  .container {
    width: 60%;
    height: 60%;
    background-color: white;
    border-radius: 20px;
    margin: auto;
    padding: 20px;
    display: grid;
    grid-template-columns: 1fr 1fr;
    grid-template-rows: 1fr 1fr;
    grid-gap: 15px;

    button {
      border-radius: 10px;
      width: 100%;
      height: 100%;
    }

    .red {
      background-color: #ff171a;
    }

    .green {
      background-color: #5db151;
    }

    .yellow {
      background-color: #ffde17;
    }

    .blue {
      background-color: #1388d7;
    }
  }
}

.direction {
  display: inline-block;
  position: absolute;
  vertical-align: middle;
  width: 350px;
  height: 350px;
  border: 14px solid transparent;
  border-top-color: #ffffff50;
  border-bottom-color: #ffffff50;
  border-radius: 50%;
  animation: rotate 8s linear infinite;
  transform: scale(1.2) rotateX(55deg);

  @media screen and (max-width: $mobile) {
    transform: scale(0.58) rotateX(55deg);
    animation: rotate-mobile 8s linear infinite;
  }

  &.reverse {
    animation-direction: reverse;

    &::after {
      top: 36px;
      left: 8px;
      transform: rotate(-133deg);
    }

    &::before {
      bottom: 36px;
      right: 8px;
      left: auto;
      transform: rotate(44deg);
    }
  }

  &::after,
  &::before {
    position: absolute;
    content: "";
    width: 0;
    height: 0;
    border: 20px solid transparent;
    border-bottom-color: #ffffff50;
  }

  &::after {
    top: 36px;
    right: 8px;
    transform: rotate(135deg);
  }

  &::before {
    bottom: 36px;
    left: 8px;
    transform: rotate(-45deg);
  }

  @keyframes rotate {
    to {
      transform: scale(1.2) rotateX(55deg) rotate(360deg);
    }
  }

  @keyframes rotate-mobile {
    to {
      transform: scale(0.58) rotateX(55deg) rotate(360deg);
    }
  }
}

.stack {
  position: absolute;
  left: 60px;
  bottom: 29vh;
  cursor: pointer;

  &.canDraw {
    box-shadow: 0px 0px 14px 14px #ffe23f, inset 0px 0px 3px 3px #ffe448;
  }

  .card {
    pointer-events: none;
    transform: rotate(-30deg) rotateY(20deg) rotateX(20deg) scale(0.85) !important;
    cursor: pointer;

    &.draw {
      animation: pulse infinite 3s ease;

      @keyframes pulse {
        from {
          box-shadow: 0px 0px 5px 4px #ffe23f, inset 0px 0px 3px 3px #ffe448;
        }

        50% {
          box-shadow: 0px 0px 14px 14px #ffe23f, inset 0px 0px 3px 3px #ffe448;
        }

        to {
          box-shadow: 0px 0px 3px 2px #ffe23f, inset 0px 0px 3px 3px #ffe448;
        }
      }
    }

    &:not(:first-of-type) {
      margin-left: 0;
      position: absolute;
      margin-top: -197px !important;
    }

    &:nth-of-type(6) {
      transform: rotate(-30deg) rotateY(20deg) rotateX(20deg)
        translate(-2px, 2px) scale(0.85) !important;
    }

    &:nth-of-type(5) {
      transform: rotate(-30deg) rotateY(20deg) rotateX(20deg)
        translate(-4px, 4px) scale(0.85) !important;
    }

    &:nth-of-type(4) {
      transform: rotate(-30deg) rotateY(20deg) rotateX(20deg)
        translate(-6px, 6px) scale(0.85) !important;
    }

    &:nth-of-type(3) {
      transform: rotate(-30deg) rotateY(20deg) rotateX(20deg)
        translate(-8px, 8px) scale(0.85) !important;
    }

    &:nth-of-type(2) {
      transform: rotate(-30deg) rotateY(20deg) rotateX(20deg)
        translate(-10px, 10px) scale(0.85) !important;
    }

    &:nth-of-type(1) {
      transform: rotate(-30deg) rotateY(20deg) rotateX(20deg)
        translate(-12px, 12px) scale(0.85) !important;
    }
  }
}

.cards {
  display: flex;
  flex-direction: row;
  margin-bottom: 50px;
  margin-top: auto;

  &.you {
    @media screen and (max-width: $mobile) {
      transform: scale(0.5);
      transform-origin: bottom center;
      margin-bottom: 30px;
    }
  }
}

.pile {
  width: 300px;
  height: 300px;
  display: flex;
  justify-content: center;
  align-items: center;
  transform: rotateX(55deg);
  position: absolute;

  @media screen and (max-width: $mobile) {
    transform: scale(0.4) rotateX(55deg);
  }
}

.you {
  .card {
    filter: brightness(0.7);
  }
}

.other {
  position: absolute;

  .card {
    pointer-events: none;
  }

  &.right {
    right: 130px;
    bottom: 47.6%;

    @media screen and (max-width: $mobile) {
      transform: scale(0.6);
      transform-origin: bottom center;
      right: 50px;
      bottom: 31%;
    }

    .card {
      transform: rotateX(25deg) rotateY(52deg) scale(0.66);

      &:not(:first-of-type) {
        margin-left: -95px;
      }
    }
  }

  &.left {
    left: 100px;
    top: 21.9%;

    @media screen and (max-width: $mobile) {
      transform: scale(0.6);
      transform-origin: top center;
      left: 50px;
      top: 17%;
    }

    .card {
      transform: rotateX(-25deg) rotateY(52deg) scale(0.66);

      &:not(:first-of-type) {
        margin-left: -95px;
      }
    }
  }

  &.top {
    top: 30px;
    margin-left: -30px;

    @media screen and (max-width: $mobile) {
      transform: scale(0.6);
      transform-origin: top center;
      top: 0px;
      margin-left: -15px;
    }

    .card {
      transform: rotateX(-30deg) scale(0.65);

      &:not(:first-of-type) {
        margin-left: -95px;
      }
    }
  }
}
</style>
