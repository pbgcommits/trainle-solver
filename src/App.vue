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
        span 🚂
      v-sheet.my-5
      span
        v-radio-group(inline v-model="mode")
          v-radio(label="Get hint" value="hint")
          v-radio(label="Solve" value="calc")
          v-radio(label="Find distance" value="two")
          v-radio(label="Determine optimal starting point" value="optimise")
          v-radio(label="Find games for station" value="game")
          v-btn(style="margin: 10px" @click="resetGuesses") Reset
          //- v-btn(style="margin-left:10px" @click="toggleExactDistances") Use precise distances
      span(v-if="mode==='calc'")
        .text-body-1 Solve today's Trainle! 
          p Enter the distance by train or the distance as the crow flies (or both!) and the Trainle Solver will reveal possible solutions to today's puzzle.
        v-text-field(class="textbox" label="Select a station" placeholder="Flinders Street" v-model="calcStation" :disabled="win || fail" @keyup="alert3=''" @keyup.enter="findStations"  :error-messages="alert3" autofocus)
        v-text-field(class="textbox" label="Number of stops" placeholder="8" v-model="calcStopDistance" :disabled="win || fail" @keyup="alert4=''" @keyup.enter="findStations"  :error-messages="alert4")
        v-text-field(class="textbox" label="Crow flies (km)" placeholder="5" v-model="calcCrowFlies" :disabled="win || fail" @keyup="alert4=''" @keyup.enter="findStations"  :error-messages="alert4")
        v-btn(style="margin: auto; width: 100%" @click="findStations" :disabled="!calcStation || !(calcStopDistance || calcCrowFlies) || win || fail") Find possible stations
        v-expand-transition
          v-table(v-if="calcGuesses.length")
            thead
              tr
                th Station 1
                th Stops apart
                th Crow flies
                th Possible station

            tbody.guesses
              tr(v-for="station in [...calcGuesses].reverse()" :key="station.station")
                th
                  .guess-station {{ station.guessedStation }}
                td
                  .guess {{ actionSymbol(station.stopDistance) }}&nbsp;{{ station.stopDistance }} {{ station.stopDistance === 1 ? 'stop' : 'stops' }}
                td
                  .guess {{ checkIfRounded(station.crowFlies) }} {{ station.crowFlies === "(No data)" ? "" : "km" }}
                td
                  .guess {{ station.station }}
      span(v-else-if="mode==='optimise'")
        .text-body-1 Find the optimal starting station for Trainle!
        p The table will display the stations and how many non-unique stations it has. 
        v-text-field(class="textbox" label="(Optional) Max non-unique stations (Default 30)" placeholder="30" v-model="maxLength" @keyup="alert4=''" @keyup.enter="findOptimalStartingPoint"  :error-messages="alert4")
        v-btn(style="margin: auto; width: 100%" @click="findOptimalStartingPoint" ) Find optimal starting point
        v-expand-transition
          v-table(v-if="Object.keys(optimalStations).length")
            thead
              tr
                th Station
                th Number of non-unique stops
                th Stations that aren't unique

            tbody.guesses
              tr(v-for="station in [...Object.keys(optimalStations).sort((a,b) => Object.keys(optimalStations[a]).length - Object.keys(optimalStations[b]).length)]")
                td
                  .guess-station {{ station }}
                td
                  .guess {{ Object.keys(optimalStations[station]).length }}
                td
                  v-table 
                    thead
                      tr
                        th Station 
                        th Stops apart
                        th Crow flies
                    tbody.info
                      tr(v-for="item in [...Object.keys(optimalStations[station])]")
                        td
                          .info {{ optimalStations[station][item]["station"] }}
                        td
                          .info {{ optimalStations[station][item]["numStops"] }} stops
                        td
                          .info {{ optimalStations[station][item]["crowFlies"] }} km
      span(v-else-if="mode==='two'")
        .text-body-1 Calculate the distance between two stations.
        v-text-field(class="textbox" label="Select a station" placeholder="Flinders Street" v-model="station1" :disabled="win || fail" @keyup="alert1=''" @keyup.enter="makeGuess"  :error-messages="alert1" autofocus)
        v-text-field(class="textbox" label="Select a second station" placeholder="Flinders Street" v-model="station2" :disabled="win || fail" @keyup="alert2=''" @keyup.enter="makeGuess"  :error-messages="alert2" )
        v-btn(style="margin: auto; width: 100%" @click="makeGuess" :disabled="!station1 || !station2 || win || fail") Calculate distance
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
      span(v-else-if="mode==='hint'")
        .text-body-1 Get a hint! 
          p Enter the distance by train or the distance as the crow flies (or both).
        v-text-field(class="textbox" label="Select a station" placeholder="Flinders Street" v-model="hintStation" :disabled="win || fail" @keyup="alert3=''" @keyup.enter="findStations"  :error-messages="alert3" autofocus)
        v-text-field(class="textbox" label="Number of stops" placeholder="8" v-model="hintStopDistance" :disabled="win || fail" @keyup="alert4=''" @keyup.enter="findStations"  :error-messages="alert4")
        v-text-field(class="textbox" label="Crow flies (km)" placeholder="5" v-model="hintCrowFlies" :disabled="win || fail" @keyup="alert4=''" @keyup.enter="findStations"  :error-messages="alert4")
        v-btn(style="margin: auto; width: 100%" @click="findStations" :disabled="!hintStation || !(hintStopDistance || hintCrowFlies) || win || fail") Get hint
        v-expand-transition
          v-table(v-if="hintGuesses.length")
            thead
              tr
                th Station 1
                th Stops apart
                th Crow flies
                th Possible lines
            // This table should only ever have one row
            tbody.guesses
              tr(v-for="station in [...hintGuesses].reverse()" :key="station.station")
                th
                  .guess-station {{ station.guessedStation }}
                td
                  .guess {{ actionSymbol(station.stopDistance) }}&nbsp;{{ station.stopDistance }} {{ station.stopDistance === 1 ? 'stop' : 'stops' }}
                td
                  .guess {{ checkIfRounded(station.crowFlies) }} {{ station.crowFlies === "(No data)" ? "" : "km" }}
                td
                  .guess {{ hintLines.toString().replaceAll(",", ", ") }}
      span(v-else-if="mode==='game'")
        .text-body-1 Find a game for a station!
          p Enter the station to find the most recent games which had it as the solution.
        v-text-field(class="textbox" label="Select a station" placeholder="Flinders Street" v-model="prevStation" :disabled="win || fail" @keyup="alert3=''" @keyup.enter="findGame"  :error-messages="alert3" autofocus)
        span
          v-radio-group(inline v-model="prevGameMode")
            v-radio(label="All previous games" value="allGames")
            v-radio(label="Previous game" value="oneGameOnly")
        //- v-text-field(class="textbox" label="Number of previous games" placeholder="1" v-model="prevStation" :disabled="win || fail" @keyup="alert3=''" @keyup.enter="findGame"  :error-messages="alert3" autofocus)
        //- v-text-field(class="textbox" label="Number of stops" placeholder="8" v-model="hintStopDistance" :disabled="win || fail" @keyup="alert4=''" @keyup.enter="findStations"  :error-messages="alert4")
        //- v-text-field(class="textbox" label="Crow flies (km)" placeholder="5" v-model="hintCrowFlies" :disabled="win || fail" @keyup="alert4=''" @keyup.enter="findStations"  :error-messages="alert4")
        v-btn(style="margin: auto; width: 100%" @click="findGame" :disabled="!prevStation || win || fail") Find game
        v-expand-transition
          v-table(v-if="prevGame.length")
            thead
              tr
                th Station
                th Game number
            // This table should only ever have one row
            tbody.guesses
              tr(v-for="game in prevGame")
                th
                  .guess-station {{ game.station }}
                td
                  .guess {{ game.gameNumber }}
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
  toggleExactDistances,
  stopDistance as stopDistanceFunc,
  // targetForGameNumber
} from "./stations";
export default {
  data: () => ({
    target: null,
    station1: "",
    station2: "",
    calcStation: "",
    calcStopDistance: "",
    calcCrowFlies: "",
    hintStation: "",
    hintStopDistance: "",
    hintCrowFlies: "",
    currentGuess: "",
    guesses: [],
    calcGuesses: [],
    hintGuesses: [],
    hintLines: [],
    optimalStations: {},
    hints: [],
    // prevGame data
    prevGame: [],
    prevStation: "",
    prevGameMode: "allGames",
    alert1: "",
    alert2: "",
    alert3: "",
    alert4: "",
    win: false,
    fail: false,
    gameNumber: 0,
    hintsLeft: undefined,
    actions: [],
    hintsAllowed: 3,
    hintBoxShowing: false,
    distancesKey: 0,
    mode: "hint",
    maxLength: ""
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
      let s = guess.toLowerCase().trim();
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
      const station1 = this.normalizeRawGuess(this.station1);
      const station2 = this.normalizeRawGuess(this.station2);
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
    findStations() {
      let stationBox;
      let stopDistanceBox;
      let crowFliesBox;
      let infoList;
      if (this.mode === "calc") {
        stationBox = this.calcStation;
        stopDistanceBox = this.calcStopDistance;
        crowFliesBox = this.calcCrowFlies;
        infoList = this.calcGuesses;
      }
      else {
        this.hintGuesses = []; // this is to only show one hint at a time
        stationBox = this.hintStation;
        stopDistanceBox = this.hintStopDistance;
        crowFliesBox = this.hintCrowFlies;
        infoList = this.hintGuesses;
      }
      const station = this.normalizeRawGuess(stationBox);
      if (!stationNames.includes(station)) {
        window.track({
          id: "invalid-guess",
          parameters: { station },
        });
        this.alert3 = "Invalid station name!";
        return;
      }
      // Ideally we would only make the user input one of these; however I'm too lazy for now
      if (!stopDistanceBox && !crowFliesBox) {
        // window.track({
        //   id: "invalid-guess",
        //   parameters: { calcStopDistance, calcCrowFlies }
        // });
        this.alert4 = "Must input at least one of the number of stops or the crow flies distance!";
        return;
      }
      if (stopDistanceBox && isNaN(stopDistanceBox) || crowFliesBox && isNaN(crowFliesBox)) {
        this.alert4 = "Numbers only!";
        return;
      }
      let infoGiven = 0;
      if (stopDistanceBox && crowFliesBox) {
        infoGiven = 3;
      }
      else if (stopDistanceBox) {
        infoGiven = 2;
      }
      else if (crowFliesBox) {
        infoGiven = 1;
      }
      const desiredStopDistance = stopDistanceBox || "(No data)";
      const desiredCrowFlies = crowFliesBox || "(No data)";
      if (infoList.length === 0) {
        window.track({
          id: "first-guess",
          parameters: { station },
        });
      }
      this.alert = "";
      // Brute force it
      let validOptionFound = false;
      for (const station2 of stationNames) {
        switch (infoGiven) {
          case 1:
            if (Math.round(stationDistance(station, station2)) === Number(desiredCrowFlies)) {
              infoList.push({
                guessedStation: stationByName(station).properties.nameUp,
                station: stationByName(station2).properties.nameUp,
                stopDistance: stopDistanceFunc(station, station2),
                crowFlies: desiredCrowFlies,
                lines: stationByName(station2).properties.lines
              });
              validOptionFound = true;
            }
            break;
          case 2:
            if (stopDistanceFunc(station, station2) === Number(desiredStopDistance)) {
              infoList.push({
                guessedStation: stationByName(station).properties.nameUp,
                station: stationByName(station2).properties.nameUp,
                stopDistance: desiredStopDistance,
                crowFlies: stationDistance(station, station2),
                lines: stationByName(station2).properties.lines
              });
              validOptionFound = true;
            }
            break;
          default:
            if (stopDistanceFunc(station, station2) === Number(desiredStopDistance) && Math.round(stationDistance(station, station2)) === Number(desiredCrowFlies)) {
            infoList.push({
              guessedStation: stationByName(station).properties.nameUp,
              station: stationByName(station2).properties.nameUp,
              stopDistance: desiredStopDistance,
              crowFlies: desiredCrowFlies,
              lines: stationByName(station2).properties.lines
            });
            validOptionFound = true;
          }
        }
      }
      if (!validOptionFound) {
        infoList.push({
          guessedStation: stationByName(station).properties.nameUp,
          station: "No valid stations were found.",
          stopDistance: desiredStopDistance,
          crowFlies: desiredCrowFlies,
          lines: []
        });
      }
      if (this.mode === "hint") {
        this.calculateHint(infoGiven);
      }
      this.currentGuess = "";
      if (this.mode === "calc") {
        this.calcStation = "";
        this.calcStopDistance = "";
        this.calcCrowFlies= "";

      }
      else if (this.mode === "hint") {
        this.hintStation = "";
        this.hintStopDistance = "";
        this.hintCrowFlies = "";

      }
    },
    calculateHint(infoGiven) {
      this.hintLines = [];
      // only show one hint
      let guess = this.hintGuesses[0];
      if (guess.lines.length === 0) {
        console.log("No valid stations!!!")
        this.hintLines[0] = "No possible stations exist."
      }
      switch (infoGiven) {
        case 1:
          guess.stopDistance = "(No data)";
          break;
        case 2:
          guess.crowFlies = "(No data)";
          break;
      }
      for (const station of this.hintGuesses) {
        for (const line of station.lines) {
          const upperLine = this.titleCase(line);
          if (!this.hintLines.includes(upperLine)) {
            console.log(upperLine)
            this.hintLines.push(upperLine);
          }
        }
      }
      this.hintGuesses = [];
      this.hintGuesses.push(guess);
      if (guess.lines.length === 0) {
        return;
      }
    },
    findOptimalStartingPoint() {
      const results = {};
      for (const stationA of stationNames) {
        for (const stationB of stationNames) {
          if (stationA === stationB) continue;
          const numStops = stopDistanceFunc(stationA, stationB);
          const crowFlies = Math.round(stationDistance(stationA, stationB));
          if (results[stationA] && results[stationA][numStops] && results[stationA][numStops][crowFlies]) {
            results[stationA][numStops][crowFlies][stationB] = 1;
            for (const key2 in results[stationA][numStops][crowFlies]) {
              if (key2 === stationB) {
                results[stationA][numStops][crowFlies][key2] += 1;
              }
            }
            continue;
          };
          if (!results[stationA]) results[stationA] = {};
          if (!results[stationA][numStops]) results[stationA][numStops] = {};
          if (!results[stationA][numStops][crowFlies]) results[stationA][numStops][crowFlies] = {};
          results[stationA][numStops][crowFlies][stationB] = 1;
        }
      }
      const finale = {};
      const maxLength = this.maxLength || 30;
      for (const stationA in results) {
        for (const numStops in results[stationA]) {
          for (const crowFlies in results[stationA][numStops]) {
            if (Object.keys(results[stationA][numStops][crowFlies]).length > 1) {
              if (!finale[stationA]) {
                finale[stationA] = [];
              }
              for (const stationB in results[stationA][numStops][crowFlies]) {
                finale[stationA].push({
                  numStops,
                  crowFlies,
                  station: stationByName(stationB).properties.nameUp
                });
              }
            }
          }
        }
        if (finale[stationA].length > maxLength) {
          delete finale[stationA];
        }
      }
      console.log(finale);
      for (const station in finale) {
        this.optimalStations[stationByName(station).properties.nameUp] = finale[station];
      }
      return finale;
    },
    /** Find the previous game(s) which had this.prevStation as the answer. */
    findGame() {
      this.prevGame = []
      let i = this.getGameNumber();
      let station = this.normalizeRawGuess(this.prevStation);
      while (i > 0) {
        if (targetForGameNumber(i) == station) {
          this.prevGame.push({"station": station, "gameNumber": i});
          this.prevStation = "";
          if (this.prevGameMode == "oneGameOnly") return;
        }
        i--;
      }
      if (!this.prevGame.length) {
        this.prevGame.push({"station": station, "gameNumber": "This station has never been an answer!"})
        this.prevStation = "";
      }
    },
    resetGuesses() {
      switch (this.mode) {
        case "two":
          this.guesses = [];
          break;
        case "calc":
          this.calcGuesses = [];
          break;
        case "hint":
          this.hintGuesses = [];
          this.hintLines = [];
          break;
        case "optimise":
          this.optimalStations = {};
          break;
      }
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
  if (isNaN(num)) {
    return num;
  }
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
