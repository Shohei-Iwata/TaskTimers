<template>
  <div class="wrapper">
    <div class="container">
      <draggable
        v-model="timers"
        item-key="id"
        tag="transition-group"
        handle=".mainTimer_sortBar"
        v-bind="dragOptions"
      >
        <template #item="{ element }">
          <div class="timerCard">
            <div class="mainTimer">
              <div class="mainTimer_sortBar"></div>
              <input class="mainTimer_name" :placeholder="element.defaultName" v-model="element.name">
              <button class="mainTimer_playButton" v-on:click="togglePlay(element)" :class="{'-playing': element.isPlay}"></button>
              <button class="mainTimer_lapButton" v-on:click="lap(element)">Lap</button>
              <button class="mainTimer_resetButton" v-on:click="reset(element)">Reset</button>
              <p class="mainTimer_totalTime timeFrame" :class="{'-red': element.isPlay}">{{ formatedTime(element.mainTime.elapsed) }}</p>
              <button class="mainTimer_deleteButton" v-on:click="remove(element)"></button>
            </div>
            <transition-group tag="div">
              <div class="lapTimer" v-for="subTimer in element.subTimers" :key="subTimer.id">
                <input class="lapTimer_name" :placeholder="subTimer.defaultName" v-model="subTimer.name">
                <p class="lapTimer_lapTime timeFrame">{{ formatedTime(subTimer.time) }}</p>
                <button class="lapTimer_deleteButton" v-on:click="removeLap(subTimer, element)"></button>
              </div>
            </transition-group>
          </div>
        </template>
      </draggable>
    </div>
    <div class="buttonWrapper">
      <div class="buttonWrapper-inner">
        <div class="newCard" v-on:click="addCard()"></div>
        <div class="reloadCards" v-on:click="resetAll()"></div>
        <div class="removeCards" v-on:click="removeAll()"></div>
      </div>
    </div>
  </div>
</template>

<script>
import draggable from 'vuedraggable';
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
export default {
  name: 'MainTimer',
  components: {
    draggable
  },
  data: function() {
    return {
      timers: [],
      count: 0,
      subCount: 0,
      draggableOptions: {
        animation: 200,
      }
    }
  },
  computed: {
    formatedTime: function() {
      return function(time) {
        return formatTime(time)
      }
    },
    dragOptions() {
      return {
        animation: 200,
        ghostClass: "dragElem"
      };
    }
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
      const title = timerName + '：' + formatTime(time.elapsed);
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
      newSubTimer.defaultName = 'ラップ' + twoDigits(newSubTimer.id + 1);
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
      const index = returnIndex(timer, this.timers);
      this.stop(timer);
      this.timers.splice(index, 1);
    },
    removeLap: function(subTimer, timer) {
      const index = returnIndex(subTimer, timer.subTimers);
      timer.subTimers.splice(index, 1);
    },
    removeAll: function() {
      if (this.timers.length === 0) return;
      const confirm = window.confirm('全てのタイマーを削除してよろしいですか？');
      if (confirm) {
        this.timers.forEach((timer) => {
          this.stop(timer);
        });
          this.timers.length = 0;
      }
    },
    addCard: function() {
      const newTimer = copyObject(defaultTimer);
      newTimer.id = this.count++;
      newTimer.defaultName = 'タイマー' + twoDigits(newTimer.id + 1)
      this.timers.push(newTimer);
    },
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
        timer.defaultName = 'タイマー' + twoDigits(timer.id + 1);
        const time = timer.mainTime;
        time.toAdd = time.elapsed;
        timer.subTimers.forEach(subTimer => {
          subTimer.id = this.subCount++;
          subTimer.defaultName = 'ラップ' + twoDigits(subTimer.id + 1);
        })
        timer.isPlay = false;
      })
    } else {
      this.addCard();
    }
  },
}
function copyObject(object) {
  return JSON.parse(JSON.stringify(object));
}
function twoDigits(number) {
  return ('00' + number).slice(-2);
}
function returnIndex (timer, timers) {
  return timers.findIndex(o => {
    return o.id == timer.id;
  })
}
function formatTime(time) {
  let h = Math.floor(time / 3600000);
  let m = Math.floor(time % 3600000 / 60000);
  let s = Math.floor(time % 60000 / 1000);
  h = ('0' + h).slice(-2);
  m = ('0' + m).slice(-2);
  s = ('0' + s).slice(-2);
  return h + ':' + m + ':' + s;
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
  &::placeholder {
    color: #bbb;
  }
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
}
.timeBox {
  width: 5rem;
}
.mainTimer {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  @include card;
  padding: 10px 15px 10px 8px;
  > *:not(:first-child):not(:last-child) {
    margin-right: 20px;
  }
}
.dragElem {
  opacity: 0.5;
}
.mainTimer_sortBar {
  height: 30px;
  width: 20px;
  margin-right: 10px;
  cursor: move;
  background: url(../assets/img/drag.svg);
  background-repeat: no-repeat;
  background-position: center;
  background-size: contain;
}
.mainTimer_playButton {
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
.mainTimer_totalTime {
  color: #000;
  &.-red {
    color: #f33;
  }
}
.mainTimer_deleteButton {
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
  > *:not(:last-child) {
    margin-right: 20px;
  }
}
.lapTimer_deleteButton {
  height: 25px;
  width: 25px;
  background: url(../assets/img/delete.svg);
  background-repeat: no-repeat;
  background-position: center;
  background-size: 60%;
}
.buttonWrapper {
  width: 635px;
  padding: 20px 20px;
  position: relative;
}
.buttonWrapper-inner {
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
  transform: rotate(0deg) translate3d(0);
}
.v-subtimer-enter-to,
.v-subtimer-leave-from {
  opacity: 1;
  height: 54px;
  margin-top: 0;
}
</style>
