<script setup>
window.g = getShortestPath;
window.toggleExactDistances = toggleExactDistances;
window.exactDistances = false;
// import Map from "./components/Map.vue";
// import HintMap from "./components/HintMap.vue";
import { stationByName } from "./stations";
import { targetForGameNumber } from "./utils.ts";
</script>

<template lang="pug">
v-app
  v-container.px-2.px-sm-4.px-md-6
    //- .pa3.flex.flex-column(style="font-family:sans-serif; height:100vh; ")
    div(style="flex-grow:1")
      h1.text-h4.text-lg-h2.text-center 🚂 Melbourne Train Station Distance Calculator
        span(v-if="isUnlimited()")  unlimited
        //- span(v-else)  \#{{ gameNumber }}
        span 🚂
      v-sheet.my-5
        .text-body-1 Calculate the distance between two stations.
          //- v-btn(style="margin-left:10px" @click="toggleExactDistances") Use precise distances
        //- .text-body-1.mt-4 Each guess reveals how many stations to the target, and the distance as the crow flies.

        //- .text-body-1.mt-4(v-if="isUnlimited()") Refresh the page to get a new target station.

      //- div.py-3(v-if="showHintMap" :style="{display:'flex',justifyContent:'center'}")
      //-   div(v-if=" target" :style="{position:'relative', height:'200px', width: playing ? '200px' : '100%'}")
      //-     HintMap(:center="stationByName(target).geometry.coordinates" :reveal="win || fail" :target="target")
      //- v-row(style="display:flex;")
      v-text-field(class="textbox" label="Select a station" placeholder="Flinders Street" v-model="station1" :disabled="win || fail" @keyup="alert1=''" @keyup.enter="makeGuess"  :error-messages="alert1" autofocus)
      v-text-field(class="textbox" label="Select a second station" placeholder="Flinders Street" v-model="station2" :disabled="win || fail" @keyup="alert2=''" @keyup.enter="makeGuess"  :error-messages="alert2" )
      v-btn(style="margin: auto; width: 100%" @click="makeGuess" :disabled="!station1 || !station2 || win || fail") Calculate distance
      //- .breaker
      //-   br
      //- a.text-body-2.ml-4.mb-4(v-if="guesses.length && playing" href="#" style="display:block;margin-top:-8px" @click.prevent="needHint" ) Need a hint?
      v-expand-transition
        v-table(v-if="guesses.length")
          thead
            tr
              th Station 1
              th Station 2
              th Stops apart
              th Crow flies
              th Distance along track

          tbody.guesses
            tr(v-for="(station1,station2) in [...guesses].reverse()" :key="station1.station")
              th
                .guess-station {{ station1.station1Up }}
              td
                .guess-station {{ station1.station2Up }}
              td
                .guess {{ actionSymbol(station1.stopDistance) }}&nbsp;{{ station1.stopDistance }} {{ station2.stopDistance === 1 ? 'stop' : 'stops' }}
              // this isn't working :(
              //- td(:key="window.exactDistances")
              td
                .guess {{ checkIfRounded(station1.distance) }} km
              //- td(:key="window.exactDistances")
              td
                .guess ~ {{ checkIfRounded(station1.distanceAlongLine) }} km
      //- #hint-box
      //- v-expand-transition
      //-   v-card.mt-12.mb-12.bg-blue-lighten-6(v-show="guesses.length && hintBoxShowing && (hints.length || playing)")
      //-     v-card-text
      //-       v-table.bg-blue-lighten-5(v-if="hints.length" density="compact")
      //-         tbody
      //-           tr(v-for="(hint,i) in hints" :key="hint")
      //-             td Hint&nbsp;{{ i+1 }}
      //-             td
      //-               span(style="font-family:Lucida Console,courier new,monospace") {{ hint }}

      //-       p(v-else) Each hint gives you clues about the names of stations near your closest guess.
      //-     v-card-actions(v-if="playing")
      //-       v-btn(v-if="hintsLeft > 0 && guesses.length" variant="tonal" @click="hint") Hint ({{ hintsLeft }} left)
      //-       v-btn.bg-blue-lighten-4(v-if="hintsLeft === 0 && playing" type="submit" @click="giveup") Give up

      //- template(v-if="win || fail")
      //-   v-card.elevation-8.my-2( style="border: 1px solid hsl(180,0%,70%)" :title="`${win ? 'Yes! ' : ''}The station is ${ titleCase(target)}.`" :class="{ 'bg-green-lighten-5': win, 'bg-red-lighten-5': fail }")
      //-     v-card-text
      //-       div(style="display: flex; flex-direction:row")
      //-         div.pr-2(style="flex-grow: 1" v-html="shareText.replace(/\\n/g,'<br>')")
      //-         v-btn.my-4.bg-white(@click="copyWin" style="flex-grow:0") Copy
      //- .map-container(v-if="guesses.length" style="width:100%; height:calc(max(50vh, 200px)); position:relative")
      //-   Map( :guesses="guesses.map(g=>g.station)" :target="target")

      //- #game-over
      //- v-btn#restart(v-if="isUnlimited() && (fail || win)" type="submit" @click="restart") Play again
    div(style="height:80px")
    v-bottom-navigation
      v-footer
        div(style="width:100%;display:flex; justify-content:space-between")
          p Made by <a href="https://github.com/pbgcommits"> Patrick Barton Grace</a>.
          p <a href="https://stevage.github.io/trainle">Trainle</a>, as well as most of the code for this project, was made by <a href="https://hire.stevebennett.me"> Steve Bennett</a>.

</template>

<script>
import confetti from "canvas-confetti";
import {
  getShortestPath,
  stationNames,
  stationDistance,
  hintForStation,
  stationByName,
  getDistanceAlongLine,
  toggleExactDistances
} from "./stations";
export default {
  data: () => ({
    target: null,
    station1: "",
    station2: "",
    currentGuess: "",
    guesses: [],
    hints: [],
    alert1: "",
    alert2: "",
    win: false,
    fail: false,
    gameNumber: 0,
    hintsLeft: undefined,
    actions: [],
    hintsAllowed: 3,
    hintBoxShowing: false,
    distancesKey: 0,
  }),
  created() {
    window.app = this;
    this.restart();
  },

  methods: {
    titleCase(s) {
      return s
        .split(" ")
        .map((w) => w[0].toUpperCase() + w.slice(1))
        .join(" ");
    },
    actionSymbol(stopDistance) {
      if (stopDistance > 20) {
        return "🟥";
      } else if (stopDistance > 5) {
        return "🟧";
        // } else if (stopDistance === 0) {
        //   return "🎉";
      } else {
        return "🟩";
      }
    },
    normalizeRawGuess(guess) {
      console.log('1.5')
      let s = guess.toLowerCase().trim();
      console.log('1.6')
      s =
        {
          jolimont: "jolimont-mcg",
          "flinders st": "flinders street",
          macauley: "macaulay",
          jewel: "jewell",
          ansty: "anstey",
          "spencer st": "southern cross",
          "spencer street": "southern cross",
        }[s] || s;
      return stationByName(s)?.properties?.name || s;
    },
    makeGuess() {
      console.log('1')
      const station1 = this.normalizeRawGuess(this.station1);
      console.log('2')
      const station2 = this.normalizeRawGuess(this.station2);
      console.log('3')
      if (station1 === station2) {
        this.alert1 = "You can't find the distance between a station and itself!";
        return;
      }
      if (!stationNames.includes(station1) && !stationNames.includes(station2)) {
        window.track({
          id: "invalid-guess",
          parameters: { station1, station2 },
        });
        this.alert1 = "Invalid station name";
        this.alert2 = "Invalid station name";
        return;
      }
      if (!stationNames.includes(station1)) {
        window.track({
          id: "invalid-guess",
          parameters: { station1 },
        });
        this.alert1 = "Invalid station name";
        return;
      }
      if (!stationNames.includes(station2)) {
        window.track({
          id: "invalid-guess",
          parameters: { station2 },
        });
        this.alert2 = "Invalid station name";
        return;
      }
      if (this.guesses.length === 0) {
        window.track({
          id: "first-guess",
          parameters: { station1, station2 },
        });
      }
      // window.track({
      //   id: "guess",
      //   parameters: {
      //     station1,
      //     station2,
      //     guessCount: this.guesses.length,
      //     guesses: this.guesses.map((g1, g2) => g1.station1, g2.station2),
      //     actions: this.actions,
      //   },
      // });

      this.alert = "";
      const stopDistance = getShortestPath(station1, station2).length - 1;
      console.log("Stop distance: " + stopDistance);
      const d = stationDistance(station1, station2);
      // const distance = window.exactDistances ? d : Math.round(d);
      // console.log("Distance: " + distance);
      const dal = getDistanceAlongLine(station1, station2);
      // const distanceAlongLine = window.exactDistances ? dal : Math.round(dal);
      this.guesses.push({
        station1: station1,
        station2: station2,
        station1Up: stationByName(station1).properties.nameUp,
        station2Up: stationByName(station2).properties.nameUp,
        stopDistance,
        distance: d,
        distanceAlongLine: dal,
        number: this.guesses.length + 1,
      });
      this.actions.push(this.actionSymbol(stopDistance));
      // if (guess === this.target) {
      //   this.winGame();
      // }
      this.currentGuess = "";
      this.station1 = "";
      this.station2 = "";
      // this.updateCookie();
    },
    winGame() {
      this.win = true;
      this.actions.push("🎉");
      window.setTimeout(() => {
        document
          .querySelector("#game-over")
          .scrollIntoView({ behaviour: "smooth" });
      }, 200);
      window.track({
        id: "win",
        parameters: {
          guessCount: this.guesses.length,
          hintCount: this.hints.length,
          target: this.target,
          actions: this.actions,
          guesses: this.guesses.map((g) => g.station),
        },
      });
      showConfetti();
    },

    getGameNumber() {
      if (window.location.search.match(/game=\d+/)) {
        const gameNumber = +window.location.search.match(/game=(\d+)/)[1];
        console.log(gameNumber);
        if (gameNumber <= this.daysSinceStart || gameNumber > 1000) {
          return gameNumber;
        }
      } else if (this.isUnlimited()) {
        return Math.floor(Math.random() * 100000 + 1000);
      }
      return this.daysSinceStart;
    },
    isUnlimited() {
      return !!window.location.search.match(/unlimited/);
    },
    testRandom() {
      const stations = {};
      const runs = 1000;
      for (let i = 1; i < runs; i++)
        stations[targetForGameNumber(i)] =
          (stations[targetForGameNumber(i)] || 0) + 1;
      const counts = Object.values(stations)
        .map((n) => +n)
        .sort((a, b) => a - b);
      console.log(...counts.slice(0, 50), "...", ...counts.slice(-50));
      console.log("expected", runs / stationNames.length);
      console.log(`${stationNames.length - counts.length} missing`);
      console.log(Object.values(stations));
      const ss = tt; //Object.keys(stations);
      // console.log(ss);
      console.log(
        ss
          .slice(0, 300)
          .map((s, i) => ss.slice(i + 1).indexOf(s))
          .sort((a, b) => a - b)
          .join(","),
      );
    },
    restart() {
      this.win = false;
      this.fail = false;
      this.hintsLeft = this.hintsAllowed;
      this.guesses = [];
      this.hints = [];
      this.actions = [];
      this.gameNumber = this.getGameNumber();
      this.target = targetForGameNumber(this.gameNumber);
      this.sessionid = String(Math.random() + (new Date() % 86400000));
      if (window.location.hostname !== "localhostz") this.loadCookie();
    },
    giveup() {
      this.fail = true;
      this.actions.push("☠");
      this.updateCookie();
      document
        .querySelector("#game-over")
        .scrollIntoView({ behaviour: "smooth" });
      window.track({
        id: "giveup",
        parameters: {
          guessCount: this.guesses.length,
          bestDistance: this.bestGuess()?.stopDistance,
          actions: this.actions,
          target: this.target,
        },
      });
    },
    bestGuess() {
      const guessVal = (guess) =>
        guess.stopDistance * 1000 +
        Math.floor(guess.distance) -
        guess.number / 1000;
      const compareGuesses = (a, b) => guessVal(a) - guessVal(b);
      return [...this.guesses].sort(compareGuesses)[0];
    },
    hint() {
      const bestStation = this.bestGuess()?.station;
      const bestDistance = this.bestGuess()?.stopDistance;
      const hintText = hintForStation(bestStation, this.target, this.hintsLeft);
      this.hintsLeft -= 1;
      this.hints.push(hintText);
      this.actions.push("🛟");
      this.updateCookie();
      window.track({
        id: "hint",
        parameters: {
          bestDistance,
          bestStation,
          hintText,
          hintCount: this.hints.length,
          guessCount: this.guesses.length,
          target: this.target,
          actions: this.actions,
        },
      });
    },

    copyWin() {
      navigator.clipboard.writeText(this.shareText);
    },
    updateCookie() {
      try {
        localStorage.setItem(
          this.gameNumber,
          JSON.stringify({
            guesses: this.guesses,
            hints: this.hints,
            actions: this.actions,
            hintsLeft: this.hintsLeft,
            win: this.win,
            fail: this.fail,
            gameNumber: this.gameNumber,
            sessionid: this.sessionid,
          }),
        );
      } catch (e) {
        console.log(e);
      }
    },
    loadCookie() {
      try {
        const cookie = localStorage.getItem(this.gameNumber);
        if (cookie) {
          const data = JSON.parse(cookie);

          this.guesses = data.guesses;
          this.hints = data.hints;
          this.actions = data.actions;
          this.hintsLeft = data.hintsLeft;
          this.win = data.win;
          this.fail = data.fail;
          this.sessionid = data.sessionid;
        } else {
          window.track({ id: "pageview", parameters: {} });
        }
      } catch (e) {
        console.log(e);
        window.track({ id: "pageview", parameters: {} });
      }
    },
    validateStation(v) {
      return !v || stationNames.includes(v.toLowerCase().trim());
    },
    needHint() {
      this.hintBoxShowing = true;
      document
        .querySelector("#hint-box")
        .scrollIntoView({ behaviour: "smooth" });
    },
  },
  computed: {
    showHintMap() {
      return window.location.search.match(/startmap/);
    },
    playing() {
      return !this.win && !this.fail;
    },
    daysSinceStart() {
      const midnightOfToday = new Date();
      midnightOfToday.setHours(0, 0, 0, 0);
      const midnightOfStart = new Date("2023-09-19");
      midnightOfStart.setHours(0, 0, 0, 0);

      const diff = midnightOfToday.getTime() - midnightOfStart.getTime();
      return Math.floor(diff / (1000 * 60 * 60 * 24)) + 1;
    },
    shareText() {
      return `I ${this.win ? "solved" : "gave up on"} 🚂Trainle #${
        this.gameNumber
      } ${this.win ? "in" : "after"} ${this.guesses.length} guesses${
        this.hintsLeft < this.hintsAllowed
          ? " with " + (this.hintsAllowed - this.hintsLeft) + " hints"
          : ""
      }.\n${this.actions.join("")}\n stevage.github.io/trainle${
        this.gameNumber !== this.daysSinceStart
          ? `?game=${this.gameNumber}`
          : ""
      }`;
    },
  },
  watch: {},
};

function checkIfRounded(num) {
  return window.exactDistances ? num.toFixed(2) : Math.round(num);
}

function showConfetti() {
  var defaults = {
    spread: 360,
    ticks: 100,
    gravity: 0.5,
    decay: 0.94,
    startVelocity: 15,
    colors: [
      "FFE400",
      "FFBD00",
      "E89400",
      "FFCA6C",
      "FDFFB8",
      "#ff5555",
      "#ff55ff",
    ],
    useWorker: true,
  };

  function shoot({ ...options } = {}) {
    confetti({
      ...defaults,
      ...options,
      particleCount: 40,
      scalar: 1.2,
      shapes: ["star"],
    });

    confetti({
      ...defaults,
      ...options,
      particleCount: 10,
      scalar: 0.75,
      shapes: ["circle"],
    });
  }

  setTimeout(shoot, 0);
  setTimeout(shoot, 200);
  setTimeout(shoot, 400, { startVelocity: 20 });
  setTimeout(shoot, 600, { startVelocity: 30 });
}

window.track = ({ id, parameters }) => {
  const url =
    window.location.hostname === "localhost" &&
    !window.location.search.match(/glitch/)
      ? "http://localhost:3080/event"
      : "https://trainle-backend.glitch.me/event";
  const body = JSON.stringify({
    site: "melbourne",
    event: id,
    gamenumber: window.app.gameNumber,
    target: window.app.target,
    guesscount: window.app.guesses.length,
    options: {
      startmap: !!window.app.showHintMap,
      unlimited: window.app.isUnlimited(),
      dev: window.location.hostname === "localhost",
      time: String(new Date()),
    },
    sessionid: window.app.sessionid,
    data: {
      ...parameters,
    },
  });
  console.log(body);
  fetch(url, {
    method: "POST",
    body,
    headers: {
      "Content-Type": "application/json",
    },
  });
};

</script>
<style>
#hintmap .mapbox-ctrl-attrib-inner {
  opacity: 0.01;
}

#hintmap .mapboxgl-control-container {
  display: none;
}

.guess {
  animation: fadeIn 0.3s;
  transform-origin: center;
  transition-timing-function: cubic-bezier(0.25, 0.1, 0.25, 1);
  /* border: 1px solid grey; */
  display: inline-block;
  color: #444;
}

.breaker {
  padding-right: 10px;
}

.textbox {
  margin: auto;
  /* padding-right: 20px; */
  width: 100%;
}

/* .buttonn {
  padding-left: 20px;
  color: red;
} */
/*
.guess,
.guess-station {
  animation: expandIn 3s;
  max-height: 1px !important;
  height: 1px !important;
} */

.guesses tr:first-child {
  background: hsl(60, 70%, 90%);
}

.guesses tr:first-child .guess {
  color: black !important;
}

@keyframes fadeIn {
  0% {
    opacity: 0.3;
    transform: scale(0.5);
  }
  70% {
    transform: scale(1.25);
  }
  100% {
    opacity: 1;
    transform: scale(1);
  }
}
/*
@keyframes expandIn {
  0% {
    max-height: 0 !important;
    height: 0 !important;
    opacity: 0.2;
  }
  100% {
    max-height: 55px !important;
    height: 55px !important;
    opacity: 1;
  }
} */
</style>
