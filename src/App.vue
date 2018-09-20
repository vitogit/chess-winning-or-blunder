<template>
  <div id="app">
    <section class="hero is-fullheight">
      <div class="hero-head">
        <nav class="navbar">
          <div class="container">
            <div>
              <a href="/">
                <p class="title">Winning or Blunder (beta)</p>
                <p class="subtitle">Answer if it's a winning move or a blunder, from your own games.</p>
              </a>
            </div>
            <div id="navbarMenu" class="navbar-menu">
              <div class="navbar-end"></div>
            </div>
          </div>
        </nav>
      </div>
      <div class="hero-body">
        <div class="container has-text-centered">
          <div class="columns">
            <div class="column is-5">
              <chessboard :fen="currentPosition.fen" :showThreats="showThreats"></chessboard>
              <div class="has-centered-text">
                {{currentPosition.white}} VS {{currentPosition.black}}
              </div>
            </div>
            <div class="column is-7">
              <QuestionGame></QuestionGame>
            </div>
          </div>
        </div>
      </div>

      <div class="hero-foot">
        <div class="container">
          <div class="has-text-centered">
            Made by <a href="http://vitomd.com">@vitomd</a> with <a href="https://vuejs.org/">Vue.js</a> | Check all my <a href="http://vitomd.com/blog/projects/">chess related projects</a>
          </div>
        </div>
      </div>
    </section>
    <a href="https://github.com/vitogit/vue-chess-guardian"><img style="position: absolute; top: 0; right: 0; border: 0;" src="https://camo.githubusercontent.com/38ef81f8aca64bb9a64448d0d70f1308ef5341ab/68747470733a2f2f73332e616d617a6f6e6177732e636f6d2f6769746875622f726962626f6e732f666f726b6d655f72696768745f6461726b626c75655f3132313632312e706e67" alt="Fork me on GitHub" data-canonical-src="https://s3.amazonaws.com/github/ribbons/forkme_right_darkblue_121621.png"></a>              </div>
  </div>

</template>

<script>
import jQuery from 'jquery'
import Chess from 'chess.js'
import {chessboard} from 'vue-chessboard'
import 'vue-chessboard/dist/vue-chessboard.css'
import { shuffle } from './Util.js'
import {defaultPositions} from './PositionData.js'
import QuestionGame from './components/QuestionGame'
import StartModal from './components/StartModal'

export default {
  name: 'App',
  components: {
    chessboard,
    QuestionGame,
    StartModal
  },
  data () {
    return {
      currentPosition: {},
      showThreats: false,
      positionNumber: 0,
      started: false,
      isStartModalActive: true,
      positionInfo: {},
      username: ''
    }
  },
  methods: {
    nextQuestion() {
      let position = this.positions[this.positionNumber]
      this.currentPosition = position
      this.$eventHub.$emit('game-changed', position)
      this.positionNumber++
    },
    start(username){ //TODO improve this method
      if (username) {
        this.positions = this.getPositions(username)
      } else {
        this.positions = defaultPositions()
      }
      if (this.positions.length < 10 ) {
        this.promptAgain()
      } else {
        this.positionNumber = 0
        this.positions = shuffle(this.positions)
        this.nextQuestion()
      }
    },
    getPositions(username){ //TODO refactor this big method and make it async
      username = username || 'e4Guardian'
      username = username.replace(/\s/g, '').toLowerCase()
      this.username = username
      this.$toast.open(`Pulling positions from: ${username}`)
      let games = []
      jQuery.ajax({
        method: 'GET',
        url: `https://lichess.org/games/export/${username}?max=50&analysed=true&evals=true&moves=true&perfType=blitz,rapid,classical`,
        async: false,
        headers: {'Accept': 'application/x-ndjson'},
        success: function (data) {
          games = data.split("\n").filter(x => x !== "").map(x => JSON.parse(x))
          games = games.slice(0,10) //get 10 blunders for testing
        },
        error: function (error) {
          console.log('Something wrong with ajax:', error);
        }
      });
      let loadedGame = new Chess()
      let blunders = this.findBlunders(games)
      let blunderPositions = this.generateBlunderPositions(blunders)
      let tacticPositions = this.generateTacticPositions(blunders)
      let positions = shuffle(blunderPositions.concat( tacticPositions))
      console.log("positions________",positions)
      return positions
    },
    findBlunders(games) {
      let blunders = []
      for (let game of games) {
        let lastEval = 0
        for (let [index, move] of game.analysis.entries() ) {
          if (move.judgment && move.judgment.name == 'Blunder' && index > 0) {
            let prevMove = game.analysis[index-1]
            if ((prevMove.eval > -300 && prevMove.eval < 950 && move.eval < 300) ||  //for white
               (prevMove.eval < 300 && prevMove.eval > -950 && move.eval > -300 )){ // for black
              blunders.push({'game': game, 'index': index, 'eval':move.eval, 'variation': move.variation})
            }
          }
        }
      }
      return blunders
    },
    generateBlunderPositions(blunders) {
      let positions = []
      for (let [index, blunder] of blunders.entries() ) {
        let game = new Chess()
        let moves = blunder.game.moves.split(' ')
        let blunderMove = {}
        let refutationMove = {}
        let result = '1-0'
        let termination = 0
        for (let [move_index, move] of moves.entries()) {
          blunderMove = game.move(move)
          if (move_index == (blunder.index)) {
            game.undo()
            termination = blunder.game.analysis[blunder.index].eval
            termination = (typeof termination === "undefined" ? 'mate:'+blunder.game.analysis[blunder.index].mate : termination.toString())
            if ((index+1)%2 != 0) { //The blunder move was made by white. so black wins
              result = '0-1'
            }
            let nextMove = moves[move_index+1]
            if ( blunder.game.analysis[blunder.index+1] ) {
              refutationMove = blunder.game.analysis[blunder.index+1].variation || nextMove
              refutationMove = refutationMove.slice(0, 4)
            }
            break
          }
        }
        let fen = game.fen()
        let variation = blunder.variation.split(' ')
        let prevEval = 0
        if (index > 0) {
          prevEval =  blunder.game.analysis[blunder.index-1].eval 
          prevEval = typeof prevEval === "undefined" ? 'mate:'+blunder.game.analysis[blunder.index-1].mate : prevEval.toString()
        }

        // if  ( (blunderMove.color == 'w' && blunder.game.players.white.user.id.toLowerCase()  == this.username)  || 
        //       (blunderMove.color == 'b' && blunder.game.players.black.user.id.toLowerCase()  == this.username)) {

        let position = {fen: game.fen(),
                        white:blunder.game.players.white.user.id, 
                        black:blunder.game.players.black.user.id,
                        url: `http://lichess.org/${blunder.game.id}#${blunder.index}`,
                        variation: refutationMove,
                        message: `From ${prevEval} to ${termination}`,
                        blunderMove: blunderMove,
                        color: blunderMove.color, 
                        answer: 'Blunder' }
        positions.push(position)
        // }
      }
      return positions
    },
    generateTacticPositions(blunders) {
      let positions = []
      for (let [index, blunder] of blunders.entries() ) {
        let game = new Chess()
        let moves = blunder.game.moves.split(' ')
        let blunderMove = {}
        let result = '1-0'
        let termination = 0
        for (let [index, move] of moves.entries()) {
          blunderMove = game.move(move)

          if (index == blunder.index) {
            game.undo()
            termination = blunder.game.analysis[index].eval
            termination = (typeof termination === "undefined" ? 'mate:'+blunder.game.analysis[index].mate : termination.toString())
            if ((index+1)%2 != 0) { //The blunder move was made by white. so black wins
              result = '0-1'
            }
            break
          }
        }
        let prevEval = 0
        if (index > 0) {
          prevEval =  blunder.game.analysis[blunder.index-1].eval 
          prevEval = typeof prevEval === "undefined" ? 'mate:'+blunder.game.analysis[blunder.index-1].mate : prevEval.toString()
        }        
        let fen = game.fen()
        let variation = blunder.variation.split(' ').slice(0, 6);
        variation.unshift(blunderMove.san)
        game = new Chess(fen)
        let position = {fen: game.fen(),
                        white: blunder.game.players.white.user.id, 
                        black: blunder.game.players.black.user.id,
                        url: `http://lichess.org/${blunder.game.id}#${blunder.index}`,
                        variation: variation, 
                        message: `From ${prevEval} to ${termination}`,
                        blunderMove: blunderMove, 
                        color: game.turn(), 
                        answer: 'Winning'
                      }
        positions.push(position)
      }
      return positions
    },      
    promptAgain() {
      this.genericPrompt({
        title: 'ERROR',
        message: `There are not enough games (${this.positions.length}) from the selected user`,
        type:'is-danger'
      })
    },
    prompt() {
      this.genericPrompt({
        title: `What's your lichess.org username?`,
        message: 'It will pull positions from the selected user '
      })
    },
    genericPrompt({title, message, type}) {
      this.$modal.open({
        parent: this,
        component: StartModal,
        hasModalCard: true,
        props: {
          title: title,
          message: message,
          inputAttrs: {
            placeholder: 'e.g. e4Guardian',
            maxlength: 20
          },
          type: type,
          onConfirm: (data) => this.start(data.username),
          onCancel: () => this.start()
        }
      })
    }
  },
  mounted(){
    this.prompt()
  },
  created() {
    this.positions = []
    this.$eventHub.$on('paint-threats', () => {
      this.showThreats = true
    })

    this.$eventHub.$on('next-question', () => {
      this.showThreats = false
      this.nextQuestion()
    })
    this.$eventHub.$on('start-again', () => {
      this.showThreats = false
      this.positionNumber = 0
      this.positions = shuffle(this.positions)
      this.nextQuestion()
    })
  }
}
</script>

<style>
  #app {
    margin: 1vh;
  }
  .hero.is-fullheight {
    min-height: 95vh;
  }
  .cg-board-wrap {
    margin: 0 auto;
  }
  .hero-foot {
    padding-bottom: 20px;
  }
</style>
