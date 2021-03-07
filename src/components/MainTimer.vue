<template>
  <div class="wrapper">
    <div class="container">
      <transition-group tag="div">
        <div class="timerCard" v-for="(timer) in timers" :key="timer" draggable="true">
          <div class="mainTimer">
            <div class="mainTimer__sortBar"></div>
            <input class="mainTimer__name" :placeholder="timer.defaultName" v-model="timer.name">
            <button class="mainTimer__playButton" v-on:click="togglePlay(timer)" :class="{'-playing': timer.isPlay}"></button>
            <button class="mainTimer__lapButton" v-on:click="lap(timer)">Lap</button>
            <button class="mainTimer__resetButton" v-on:click="reset(timer)">Reset</button>
            <p class="mainTimer__totalTime timeFrame" :class="{'-red': timer.isPlay}">{{ formatedTime(timer.mainTime.elapsed) }}</p>
            <button class="mainTimer__deleteButton" v-on:click="remove(timer)"></button>
          </div>
          <transition-group tag="div">
            <div class="lapTimer" v-for="subTimer in timer.subTimers" :key="subTimer.id">
              <input class="lapTimer__name" :placeholder="subTimer.defaultName" v-model="subTimer.name">
              <p class="lapTimer__lapTime timeFrame">{{ formatedTime(subTimer.time) }}</p>
              <button class="lapTimer__deleteButton" v-on:click="removeLap(subTimer, timer)"></button>
            </div>
          </transition-group>
        </div>
      </transition-group>
    </div>
    <div class="button-wrapper">
      <div class="button-wrapper-inner">
        <div class="newCard" v-on:click="addCard()"></div>
        <div class="reloadCards" v-on:click="resetAll()"></div>
        <div class="removeCards" v-on:click="removeAll()"></div>
      </div>
    </div>
  </div>
</template>

<script>
const defaultTimer = {
          id: 0,
          name: "",
          defaultName: "",
          mainTime: {
            start: 0,
            elapsed: 0,
            toAdd: 0,
            lapToAdd: 0,
          },
          subTimers: [],
          isPlay: false,
          isRemoved: false,
        };
const defaultSubTimer = {
  id: 0,
  name: "",
  defaultName: "",
  time: 0,
  isRemoved: false,
}
function copyObject(object) {
  return JSON.parse(JSON.stringify(object));
}
export default {
  name: 'MainTimer',
  data: function() {
    return {
      timers: [],
      count: 0,
      subCount: 0,
    }
  },
  computed: {
    formatedTime: function() {
      return function(time) {
        return this.formatTime(time)
      }
    },
  },
  methods: {
    play: function(timer) {
      this.timers.forEach((timer) => {
        if(timer.isPlay) this.stop(timer);
      });
      timer.isPlay = true;
      timer.mainTime.start = Date.now();
      this.changeTitle(timer);
      document.getElementById("fav").href="favicon_red.svg";
      countUp();
      const self = this;
      function countUp() {
        timer.playing = setTimeout(() => {
          self.updateTime(timer);
          countUp();
        }, 1000);
      }
    },
    updateTime: function(timer) {
      const time = timer.mainTime;
      time.elapsed = Date.now() - time.start + time.toAdd;
      this.changeTitle(timer);
    },
    changeTitle: function(timer) {
      const time = timer.mainTime;
      let timerName = timer.name || timer.defaultName;
      if(timerName.length > 6) {
        timerName = timerName.substr(0, 6) + '..';
      }
      const title = timerName + '：' + this.formatTime(time.elapsed);
      document.title = title;
    },
    stop: function(timer) {
      const time = timer.mainTime;
      timer.isPlay = false;
      document.getElementById("fav").href="favicon.svg";
      clearTimeout(timer.playing);
      time.toAdd += Date.now() - time.start;
      document.title = 'tasktimers';
    },
    togglePlay: function(timer) {
      timer.isPlay? this.stop(timer): this.play(timer);
    },
    lap: function(timer) {
      const time = timer.mainTime;
      const lapTime = time.elapsed - time.lapToAdd;
      time.lapToAdd = time.elapsed;
      const newSubTimer = copyObject(defaultSubTimer);
      newSubTimer.id = this.subCount++;
      newSubTimer.defaultName = 'ラップ' + this.twoDigits(newSubTimer.id + 1);
      newSubTimer.time = lapTime;
      timer.subTimers.push(newSubTimer);
    },
    reset: function(timer) {
      if(timer.isPlay) this.stop(timer);
      timer.mainTime = copyObject(defaultTimer.mainTime);
    },
    resetAll: function() {
      this.timers.forEach(timer => {
        this.reset(timer);
        timer.subTimers.length = 0;
      })
    },
    remove: function(timer) {
      const index = this.returnIndex(timer, this.timers);
      this.stop(timer);
      this.timers.splice(index, 1);
    },
    removeLap: function(subTimer, timer) {
      const index = this.returnIndex(subTimer, timer.subTimers);
      timer.subTimers.splice(index, 1);
    },
    removeAll: function() {
      this.timers.forEach((timer) => {
        this.stop(timer);
      });
        this.timers.length = 0;
    },
    addCard: function() {
      const newTimer = copyObject(defaultTimer);
      newTimer.id = this.count++;
      newTimer.defaultName = 'タイマー' + this.twoDigits(newTimer.id + 1)
      this.timers.push(newTimer);
    },
    twoDigits: function(number) {
      return ('00' + number).slice(-2);
    },
    returnIndex: function(timer, timers) {
      return timers.findIndex(o => {
        return o.id == timer.id;
      })
    },
    formatTime: function(time) {
      let h = Math.floor(time / 3600000);
      let m = Math.floor(time % 3600000 / 60000);
      let s = Math.floor(time % 60000 / 1000);
      h = ('0' + h).slice(-2);
      m = ('0' + m).slice(-2);
      s = ('0' + s).slice(-2);
      return h + ':' + m + ':' + s;
    }
  },
  watch: {
    timers: {
      handler: function(next) {
        localStorage.setItem("lastSession", JSON.stringify(next));
      },
      deep: true
    }
  },
  mounted: function(){
    const lastSession = JSON.parse(localStorage.getItem("lastSession"));
    if(lastSession && lastSession.length > 0) {
      this.timers = lastSession;
      this.timers.forEach(timer => {
        timer.id = this.count++;
        timer.defaultName = 'タイマー' + this.twoDigits(timer.id + 1);
        const time = timer.mainTime;
        time.toAdd = time.elapsed;
        timer.subTimers.forEach(subTimer => {
          subTimer.id = this.subCount++;
          subTimer.defaultName = 'ラップ' + this.twoDigits(subTimer.id + 1);
        })
        timer.isPlay = false;
      })
    } else {
      this.addCard();
    }
  },
}
</script>

<style scoped lang="scss">
img {
  width: 100%;
  height: 100px;
}

@mixin upShadow {
  box-shadow:  1px 1px 3px #a3a8a8, -1px -1px 3px #ffffff;
}
@mixin downShadow {
  box-shadow: inset 1px 1px 3px #a3a8a8, inset -1px -1px 3px #ffffff;
}
input {
  border-radius: 5px;
  padding: 0.5rem;
  @include downShadow;
  text-align: left;
}

button {
  background: #ffffff;
  @include upShadow;
  border-radius: 5px;
  padding: 0.5rem;
  transition: 0.1s;
  &:active {
    @include downShadow;
  }
}
.wrapper {
  display: inline-flex;
  flex-direction: column;
  align-items: center;
  margin-top: 20px;
}
.container {
  display: inline-block;

}
@mixin card {
  background-color: #ffffff;
  border-radius: 15px;
  box-shadow: 5px 10px 10px -5px rgba(0,0,0,0.2);
  position: relative;
}
.timerCard {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  margin: 0 20px 20px;
  cursor: move;
}
.timeBox {
  width: 5rem;
}

.mainTimer {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  @include card;
  padding: 10px 15px;
  > *:not(:last-child) {
    margin-right: 20px;
  }
  > *:first-child {
    margin-right: 15px;
  }
}
.mainTimer__sortBar {
  display: flex;
  flex-direction: column;
  & > * {
    width: 30px;
    height: 20px;
    font-size: 12px;
  }
}
.dragElem {
  opacity: 0.5;
}
.over::before {
  content: "";
  position: absolute;
  top: -10px;
  left: 50%;
  width: 580px;
  height: 2px;
  transform: translateX(-50%);
  background: hsl(350, 100%, 80%);
}
.mainTimer__sortBar {
  height: 30px;
  width: 10px;
  margin-right: 5px;
  background: url(../assets/img/drag.svg);
  background-repeat: no-repeat;
    background-position: center;
    background-size: 80%;
}

.mainTimer__upButton {
  background: url(../assets/img/up.svg);
    background-repeat: no-repeat;
    background-position: center;
    background-size: 70%;
}
.mainTimer__downButton {
  background: url(../assets/img/down.svg);
    background-repeat: no-repeat;
    background-position: center;
    background-size: 70%;
}
.mainTimer__playButton {
  height: 35px;
  width: 35px;
  background: url(../assets/img/play.svg);
  background-repeat: no-repeat;
  background-position: 65% center;
  background-size: 60%;
  &.-playing {
    background: url(../assets/img/stop.svg);
    background-repeat: no-repeat;
    background-position: center;
    background-size: 70%;
  }
}
.mainTimer__totalTime {
  color: #000;
  &.-red {
    color: #f33;
  }
}
.mainTimer__deleteButton {
  height: 25px;
  width: 25px;
  background: url(../assets/img/delete.svg);
  background-repeat: no-repeat;
  background-position: center;
  background-size: 60%;
}

.lapTimer {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  @include card;
  padding: 10px 15px;
  margin-top: 5px;
  // animation: lapApper ease 0.3s forwards;
  > *:not(:last-child) {
    margin-right: 20px;
  }
}
// @keyframes lapApper {
//   0% {
//     transform: translateY(-10px);
//     z-index: -1;
//   }
//   100% {
//     transform: translateY(0);
//   }
// }
.lapTimer__deleteButton {
  height: 25px;
  width: 25px;
  background: url(../assets/img/delete.svg);
  background-repeat: no-repeat;
  background-position: center;
  background-size: 60%;
}
.button-wrapper {
  width: 635px;
  padding: 20px 20px;
  position: relative;
}
.button-wrapper-inner {
  display: flex;
  justify-content: space-between;
  width: (635px / 2);
  margin-left: auto;
}
@mixin bottom-button {
  height: 45px;
  width: 45px;
  @include card;
  box-shadow: 5px 10px 10px -5px rgba(0,0,0,0.2);
  border-radius: 50%;
  position: relative;
  transition: 0.2;
  cursor: pointer;
  &:active {
    box-shadow: none;
  }
  &::after {
    content: "";
    position: absolute;
    height: 35px;
    width: 35px;
    top: 50%;
    left: 50%;
    transform: translateX(-50%) translateY(-50%);
  }
}
.newCard {
  @include bottom-button;
  &::after {
    background: url(../assets/img/plus.svg);
    background-repeat: no-repeat;
    background-position: center;
    background-size: 60%;
  }
}
.reloadCards {
  @include bottom-button;
  &::after {
    background: url(../assets/img/reload.svg);
    background-repeat: no-repeat;
    background-position: center;
    background-size: 70%;
  }
}
.removeCards {
  @include bottom-button;
  margin-right: 5px;
  &::after {
    background: url(../assets/img/trash.svg);
    background-repeat: no-repeat;
    background-position: center;
    background-size: 70%;
  }
}
.deleted {
  position: relative;
  opacity: 0;
  height: 0;
  margin-top: 0;
  margin-bottom: 0;
  padding-top: 0;
  padding-bottom: 0;
  transition: 0.5s;
  z-index: -1;
}

.v-enter-from,
.v-leave-to {
  opacity: 0;
  height: 0px;
  margin-bottom: 0px;
  margin-top: 0px;
  padding-top: 0;
  padding-bottom: 0;
}
.v-enter-from {
  transform: perspective(500px) translate3d(0, 0, 50px);
}
.v-leave-to {
  transform: rotateX(90deg);
}
.v-enter-active,
.v-leave-active {
  transition: 700ms ease;
}
.v-enter-to,
.v-leave-from {
  opacity: 1;
  height: 55px;
  // margin-bottom: 20px;
  transform: rotate(0deg) translate3d(0);
}
.v-subtimer-enter-to,
.v-subtimer-leave-from {
  opacity: 1;
  height: 54px;
  margin-top: 0;
}
</style>
