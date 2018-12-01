<template>
  <div class="roulette-wrap d-flex-center">
    <div class="container margin-sides">
      <h1 class="header-txt">Aardvark Roulette game API demo</h1>
      <div class="api-group">
        <h4>API base URL</h4>
        <input id="api_base" value="https://dev-games-backend.advbet.com/v1/ab-roulette/1/">
      </div>
      <div class="stats-group">
        <h4>Stats (last 200)</h4>
        <table class="table">
          <thead>
            <tr>
              <th></th>
              <th colspan="5" class="cold">Cold</th>
              <th colspan="27" class="neutral">Neutral</th>
              <th colspan="5" class="hot">Hot</th>
            </tr>
          </thead>
          <tbody>
            <tr>
              <th>Slot</th>
              <td
                v-for="(v, i) in stats"
                :key="i"
                :class="[`btn-${config.colors[v.result]}`]">
                {{  v.result }}
              </td>
            </tr>
            <tr>
              <th>Hits</th>
              <td
                v-for="(v, i) in stats"
                :key="i"
                :class="{'cold': i < 5, 'neutral': i >= 5 && i < stats.length - 5,'hot': i >= stats.length - 5}">
                {{ v.count }}
              </td>
            </tr>
          </tbody>
        </table>
      </div>
      <div class="game-wrap">
        <div class="game-wrapper">
          <h4>Game board</h4>
          <div class="board-wrapper">
            <button
              v-for="(v, i) in config.slots"
              :key="i"
              :class="[`btn-${config.colors[config.positionToId[i]]}`]"
              :data-id="[`slot-${config.positionToId[i]}`]">
              {{ config.results[config.positionToId[i]] }}
            </button>
          </div>
          <div class="event-wrapper">
            <h4>Events</h4>
            <ul class="list-group">
              <li
                v-for="game in gameInfo"
                :key="game">
                {{ game }}
              </li>
              <li>
                {{ currentGame }}
              </li>
            </ul>
          </div>
        </div>
        <div class="log-wrapper">
          <h4>Log</h4>
          <pre><span v-for="v in logArr" :key="v">{{ v }}</span></pre>
        </div>
      </div>
    </div>
    <div v-if="spinner === true" class="spinner"></div>
  </div>
</template>

<script>
export default {
  name: 'Roulette',
  data () {
    return {
      apiUrl: '',
      firstGame: true,
      config: [],
      stats: [],
      logArr: [],
      gameInfo: [],
      currentGame: '',
      spinner: null,
      timeDelay: 1000
    }
  },
  mounted () {
    this.apiUrl = document.querySelector('#api_base').value
    this.loadBoard()
    this.newGame()
  },
  methods: {
    logEvents (data) {
      let now = new Date()
      this.logArr.push(`${now.toISOString()} ${data} ${'\r\n'}`)
    },
    startSpinner () {
      this.logEvents('Spinning the wheel')
      if (this.spinner === true) {
        this.logEvents('Wheel is already spinning &#10160;')
      }
    },
    stopSpinner () {
      this.logEvents('Stopping the wheel')
      this.spinner = false
    },
    fadeAnimation (element) {
      element.classList.add('fade-out-in')
      setTimeout(() => {
        element.classList.remove('fade-out-in')
      }, 4000)
    },
    loadBoard () {
      this.logEvents('Loading game board')
      this.logEvents('GET .../configuration')
      fetch(this.apiUrl + 'configuration')
        .then(res => res.json())
        .then(data => {
          this.config = data
        })
        .catch(err => console.log(err))
    },
    getStats () {
      fetch(this.apiUrl + 'stats?limit=200')
        .then(res => res.json())
        .then(data => {
          this.logEvents(`GET .../stats?limit=200`)
          this.stats = data
        })
        .catch(err => console.log(err))
    },
    newGame () {
      this.logEvents('Checking for new game')
      if (this.firstGame !== true) { this.getStats() }
      fetch(this.apiUrl + 'nextGame')
        .then(res => res.json())
        .then(data => {
          this.logEvents(`GET .../nextGame`)
          if (this.firstGame === true) {
            this.getStats()
            this.firstGame = false
          }
          this.waitingForGame(data)
        })
        .catch(err => console.log(err))
    },
    waitingForGame (game) {
      if (game.fakeStartDelta <= 0) {
        this.startGame(game)
      }
      this.logEvents(`sleeping for fakeStartDelta ${game.fakeStartDelta} sec`)
      this.currentGame = `Game ${game.id} will start in ${game.fakeStartDelta} sec`
      this.$options.interval = setInterval(() => {
        game.fakeStartDelta -= 1
        if (game.fakeStartDelta > 0) {
          this.currentGame = `Game ${game.id} will start in ${game.fakeStartDelta} sec`
          return
        }
        clearInterval(this.$options.interval)
        this.startGame(game)
      }, this.timeDelay)
    },
    startGame (game) {
      this.startSpinner()
      this.spinner = true
      this.logEvents(`GET .../game/${game.id}`)
      this.currentGame = `Game ${game.id}, the wheel is spinning`
      fetch(this.apiUrl + `game/${game.id}`)
        .then(res => res.json())
        .then(data => {
          if (data.result === null) {
            this.logEvents('Still no result continue spinning')
            setTimeout(() => {
              this.startGame(game)
            }, this.timeDelay)
          } else {
            this.logEvents(`Result is ${data.result}`)
            let fadeElement = document.querySelector(`button[data-id=slot-${data.result}]`)
            this.fadeAnimation(fadeElement)
            this.currentGame = `Game ${game.id} ended, result ${data.result}`
            this.gameInfo.push(this.currentGame)
            this.stopSpinner()
            this.newGame()
          }
        })
        .catch(err => console.log(err))
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.d-flex-center {
  display: flex;
  justify-content: center;
}
.container {
  max-width: 1170px;
  width: 100%;
}
.margin-sides {
  margin: 0 20px;
}

.cold {
  background-color: #D9EDF7;
}
.neutral {
  background-color: #fff;
}
.hot {
  background-color: #F2DEDE;
}

.fade-out-in {
  animation: fade 4000ms forwards;
}
@keyframes fade {
  0% {opacity: 1}
  15% {opacity: 0}
  30% {opacity: 1}
  45% {opacity: 0}
  60% {opacity: 1}
  75% {opacity: 0}
  100% {opacity: 1}
}

.btn-red {
  color: #fff;
  background-color: #D9534F;
  border-color: #AC2925;
}
.btn-black {
  color: #fff;
  background-color: #000;
  border-color: #222;
}
.btn-green {
  color: #fff;
  background-color: #449D44;
  border-color: #398439;
}

.header-txt {
  margin: 20px 0 10px 0;
  text-align: left;
}
.api-group {
  width: 100%;
  margin-bottom: 25px;
}
  .api-group > h4 {
    margin: 10px 0;
    text-align: left;
  }
  .api-group > input {
    width: 100%;
    padding: 6px 12px;
    font-size: 14px;
    line-height: 1.4;
    background-color: #fff;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

.stats-group {
  width: 100%;
}
  .stats-group > h4 {
    text-align: left;
    margin: 0 0 10px 0;
  }

  .stats-group .table {
    width: 100%;
    margin-bottom: 25px;
    border-collapse: collapse;
  }
    .table thead th {
      text-align: left;
    }
    .table tr {
      border-top: 1px solid #ddd;
    }
      .table tr > th,
      .table tr > td {
        padding: 8px;
        font-size: 14px;
      }

  .game-wrap {
    width: 100%;
    display: flex;
    flex-direction: row;
  }
    .game-wrapper {
      display: flex;
      flex-direction: column;
      width: 50%;
      margin-right: 15px;
    }
      .game-wrapper > h4 {
        text-align: left;
        margin-bottom: 10px;
      }
      .board-wrapper {
        max-width: 500px;
        width: 100%;
        display: flex;
        flex-direction: row;
        flex-wrap: wrap;
        margin-bottom: 25px;
      }
        .board-wrapper > button {
          width: 40px;
          height: 40px;
          border-radius: 4px;
          border: 1px solid #ccc;
          font-size: 14px;
          cursor: pointer;
        }
        .board-wrapper > button:hover {
          color: #666;
        }

      .game-wrapper .event-wrapper {
        width: 100%;
        display: flex;
        flex-direction: column;
        margin-bottom: 25px;
      }
        .event-wrapper > h4 {
          text-align: left;
          margin-bottom: 10px;
        }
        .event-wrapper > ul {
          list-style: none;
        }
          .event-wrapper > ul li {
            width: 100;
            display: block;
            padding: 10px 15px;
            border: 1px solid #ddd;
            text-align: left;
            margin-bottom: -1px;
          }
          .event-wrapper > ul li:first-child {
            border-top-left-radius: 4px;
            border-top-right-radius: 4px;
          }
          .event-wrapper > ul li:last-child {
            border-bottom-left-radius: 4px;
            border-bottom-right-radius: 4px;
          }

    .log-wrapper {
      width: 50%;
      margin-left: 15px;
      float: left;
    }
      .log-wrapper > h4 {
        text-align: left;
        margin-bottom: 10px;
      }
      .log-wrapper > pre {
        display: block;
        padding: 9.5px;
        margin: 0 0 25px 0;
        font-size: 13px;
        line-height: 1.4;
        color: #333;
        word-break: break-all;
        word-wrap: break-word;
        background-color: #f5f5f5;
        border-radius: 4px;
        overflow: auto;
        border: 1px solid #ccc;
        text-align: left;
      }

.spinner {
  position: absolute;
  align-self: center;
  width: 100px;
  height: 100px;
  border: 14px solid #f1dbdb;
  border-radius: 50%;
  border-top: 14px solid #23e8be;
  animation: spin 750ms linear infinite;
  -webkit-animation: spin 750ms linear infinite;
}
@keyframes spin {
  0% {transform: rotate(0deg)}
  100% {transform: rotate(360deg)}
}
@-webkit-keyframes spin {
  0% {-webkit-transform: rotate(0deg)}
  100% {-webkit-transform: rotate(360deg)}
}
</style>
