<template>
  <div>
    <section class="hero" ref="question" v-if="status != 'end'">
      <div class="hero-body">
        <div class="container">
          <h1 class="title">
            {{number}} - <span v-html="question"></span>
          </h1>
        </div>
      </div>
    </section>
    <section class="hero" ref="answers">
      <div class="buttons is-centered" v-if="!showingResult">
        <span v-html="answer" @click="verifyAnswer(answer)" class="button is-large" v-for="answer in answers">
        </span>
      </div>
      <div class="is-centered" v-if="showingResult">
        <article v-if="status == 'correct'" class="message is-success">
          <div class="message-header">
            <p>Correct</p>
          </div>
          <div class="message-body">
            Correct. It's {{this.correctAnswer=='Best move' ? 'the': 'a' }} <strong> {{this.correctAnswer}} </strong>.
            {{this.currentPosition.message}}<br>
            You can play the variation in the board: {{this.currentPosition.variation}} <br/>
            <a target="_blank"  class="button" :href="this.currentPosition.url">Analyze in lichess.org</a>
            <span class="button" @click="nextQuestion"> Next question</span>
          </div>
        </article>
        <article v-if="status == 'incorrect'" class="message is-danger">
          <div class="message-header">
            <p>Incorrect</p>
          </div>
          <div class="message-body">
            Incorrect. It's {{this.correctAnswer=='Best move' ? 'the': 'a' }} <strong> {{this.correctAnswer}} </strong><br/>
            {{this.currentPosition.message}}  <br>
            You can play the variation in the board: {{this.currentPosition.variation}} <br/>
            <a target="_blank"  class="button" :href="this.currentPosition.url">Analyze in lichess.org</a>
            <span class="button" @click="nextQuestion"> Next question</span>
          </div>
        </article>
        <article v-if="status == 'end'" class="message is-info">
          <div class="message-header">
            <p>The end</p>
          </div>
          <div class="message-body">
            You got <strong> {{this.corrects}} </strong> correct answers in {{this.maxQuestions}} questions<br/>
            Did you like it? Share it with your friends
            <span class="button" @click="startAgain"> Start Again</span>
          </div>
        </article>
      </div>
    </section>


  </div>
</template>

<script>
import { shuffle } from '../Util.js';

export default {
  name: 'QuestionGame',
  data () {
    return {
      number: 1,
      question: null,
      answers: [],
      correctAnswer: null,
      showingResult: false,
      status: null,
      corrects: 0,
      maxQuestions: 10,
      currentPosition: {}
    }
  },
  methods: {
    start(position) {
      let color = (position.color == 'w' ? 'White' : 'Black')
      this.question = `${color} to move, is <span class="highlight">${position.move.san}</span> the best move or a blunder?`
      this.correctAnswer = position.answer
      this.currentPosition = position
      this.answers = ['Best move', 'Blunder']
    },
    verifyAnswer(answer) {
      if (answer == this.correctAnswer) {
        this.status = 'correct'
        this.corrects++
      } else {
        this.status = 'incorrect'
      }
      this.showingResult = true
    },
    nextQuestion() {
      this.correctAnswer = null
      this.showingResult = false
      this.status = null
      if (this.number >= this.maxQuestions) {
        this.endGame()
      } else {
        this.$eventHub.$emit('next-question')
      }
      this.number++
    },
    playVariation() {
      
    },
    resetPosition() {
      
    },
    endGame() {
      this.showingResult = true
      this.status = 'end'
    },
    startAgain() {
      this.number= 1
      this.correctAnswer = null
      this.showingResult = false
      this.status = null
      this.corrects =  0
      this.$eventHub.$emit('start-again')
    }
  },
  created() {
    this.$eventHub.$on('game-changed', game => {
      this.start(game)
    })
  }
}
</script>
<style>
.highlight {
  color: #000;
  background-color: #ff0;
  padding: 0 5px;
}
</style>
