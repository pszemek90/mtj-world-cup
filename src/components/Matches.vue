<template>
  <div>
    <span class="overall-pool" v-show="isToday">Dzisiejsza pula: {{overallPool}}</span>
    <div class="datepicker">
      <ui-datepicker
        v-model="date"
        :config="config"
        placeholder="Select Date..."
        outlined
        toggle>
      </ui-datepicker>
    </div>
    <ui-form type="|" action-align="center">
      <template #default="{subitemClass, actionClass}">
        <div class="match" v-for="match in matches" :key="match.id">
	        <ui-checkbox v-show="!match.disabled" v-model="match.chosen" :disabled="match.disabled"></ui-checkbox>
	        <span class="team">{{match.homeTeam}}</span>
	        <ui-form-field v-show="!match.disabled">
		        <ui-textfield v-model.trim="match.homeScore" input-type="number" pattern="\d*" class="score" @input="match.chosen = true" endAligned outlined :disabled="match.disabled" helper-text-id="score-helper-text"></ui-textfield>
		        <!--            <ui-textfield-helper id="score-helper-text" :valid-msg="validMsg.homeScore"></ui-textfield-helper>-->
	        </ui-form-field>
	        -
	        <ui-form-field v-show="!match.disabled">
		        <ui-textfield input-type="number" pattern="\d*" class="score" @input="match.chosen = true" v-model.trim="match.awayScore" outlined :disabled="match.disabled" required></ui-textfield>
	        </ui-form-field>
	        <span class="team">{{match.awayTeam}}</span>
	        <div>
		        <p>g. {{matchTime(match)}}</p>
		        <p v-show="match.disabled">pula: {{match.pool}}</p>
	        </div>
        </div>
        <span class="errorMessage" v-show="errorMessage">Wystąpił błąd połączenia z serwerem. Spróbuj później. {{errorMessage}}</span>
		<span class="errorMessage" v-show="wrongTypings">{{wrongTypings}}</span>
        <ui-form-field :class="actionClass">
          <ui-button class="sendButton" @click.prevent="showSendTypingModal" raised>Wyślij</ui-button>
        </ui-form-field>
      </template>
    </ui-form>
  </div>
  <ui-dialog v-model="sendTypingsModalOpened" @confirm="sendTyping">
    <ui-dialog-title>Wysyłane typowania</ui-dialog-title>
    <ui-dialog-content>
      <p>
        Twoje typowania to:
      </p>
      <p v-for="match in sentMatches">
        {{match.homeTeam}} {{match.homeScore}} - {{match.awayScore}} {{match.awayTeam}}
      </p>
    </ui-dialog-content>
    <ui-dialog-actions></ui-dialog-actions>
  </ui-dialog>
  <ui-snackbar v-model="showSuccessSnackbar" 
    message="Pomyślnie wysłano" :action-type="actionType" position="top">
  </ui-snackbar>
  <ui-snackbar v-model="showErrorSnackbar" 
    message="Wystąpił błąd" :action-type="actionType" position="top">
  </ui-snackbar>
</template>

<script>
import axios from 'axios';
import authHeader from './../service/auth-header';
import BalmUI, {useValidator} from 'balm-ui';

/*const validations = [
  {
	key: 'homeTeam',
	label: 'Home Score',
	validator: 'required, homeTeam'
  }
]*/

  export default {
    name: 'Matches',
    components: {
      BalmUI
    },
    data() {
      return {
        matches:[],
        config: {
          defaultDate: 'today',
          minDate: 'today',
          locale: {
            "firstDayOfWeek": 1
          }
        },
        date: 'today',
        typings: {
          matches: [],
          userId: 0,
          date: 'today'
        },
        errorMessage:'',
        sendTypingsModalOpened: false,
        showSuccessSnackbar: false,
        showErrorSnackbar: false,
        actionType: 1,
        overallPool: '',
        balmUI: useValidator(),
        // validations,
        validMsg: {},
		wrongTypings: '',
		sentMatches: []
      }
    },
    computed: {
      loggedIn() {
        return this.$store.state.auth.status.loggedIn;
      },
      chosenMatches() {
        return this.matches.filter(match => match.chosen)
      },
      isToday() {
        let rawCalendarDate = new Date(this.date)
        let calendarDate = rawCalendarDate.getFullYear() + '-' + rawCalendarDate.getMonth() + '-' + rawCalendarDate.getDate()
        let rawToday = new Date()
        let today = rawToday.getFullYear() + '-' + rawToday.getMonth() + '-' + rawToday.getDate()
        return calendarDate === today
      }
    },
    methods: {
      getMatches() {
        axios.get(this.$store.state.origin + ':8080/matches/today', {
          params: {
            date: this.date
          },
          headers: authHeader() 
        })
          .then((response) => {
            this.matches = response.data
			this.matches.forEach(match => match.homeScore = '')
			this.matches.forEach(match => match.awayScore = '')
            this.markPastMatches()
            this.errorMessage = ''
          })
          .catch((error) => {
            this.errorMessage = error.message
          })
      },
      matchTime(match) {
        function padTo2Digits(num) {
          return String(num).padStart(2, '0')
        }
        let date = new Date(match.date)
	      return padTo2Digits(date.getHours()) + ':' + padTo2Digits(date.getMinutes());
      },
      showSendTypingModal() {
        /*let tmp = JSON.parse(JSON.stringify(this.chosenMatches))
        console.log(tmp[0])
        let result = this.balmUI.validate(tmp[0]);
        let {valid, validMsg} = result;
        this.validMsg = validMsg;
        if(valid) {*/
	      this.wrongTypings = ''
	      if(this.chosenMatches.length === 0) {
			  this.wrongTypings = 'Musisz wysłać przynajmniej jeden mecz.'
	      } else if(this.chosenMatches.filter(match => match.homeScore === '').length > 0
		      || this.chosenMatches.filter(match => match.awayScore === '').length > 0) {
			  this.wrongTypings = 'Nie możesz wysłać pustego wyniku'
	      } else {
			  this.sentMatches = this.chosenMatches.filter(match => Date.parse(match.date) > Date.now())
		      this.sendTypingsModalOpened = true
	      }
        // }
      },
      sendTyping(result) {
        if(result) {
          this.typings.userId = this.$store.state.auth.user.id
          axios.post(this.$store.state.origin + ':8080/matches/typings', {
            "matches": this.sentMatches,
            "userId": this.typings.userId
          }, {
            headers: authHeader()
          }).then((response) => {
            if(response) {
              this.showSuccessSnackbar = true
            }
          }).catch((error) => {
            this.showErrorSnackbar = true
          })
        }
        this.sendTypingsModalOpened = false
      },
      markPastMatches() {
        for(let i = 0; i < this.matches.length; i++) {
          let matchDate = new Date(this.matches[i].date).getTime()
          let now = Date.now()
          if(matchDate < now) {
            this.matches[i].disabled = true
          } else {
            this.matches[i].disabled = false
          }
        }
      },
      getOverallPool() {
        axios.get(this.$store.state.origin + ':8080/overallPool', {
          headers: authHeader()
        }).then((response) => {
          this.overallPool = response.data.overallPool + ' zł'
        }).catch((error) => {
          this.overallPool = 'Nie udało się pobrać puli'
        })
      }
    },
    watch: {
      date() {
        if(this.isToday) {
          this.getOverallPool()
        }
        this.getMatches()
      }
    }
  }
</script>

<style scoped>
.match {
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 8px 0;
}

.team {
  width: 100px;
  margin: 0 5px;
  text-align: center;
}
@media (min-width: 600px){
	.score {
		width: 70px;
		margin: 0 5px;
	}
}
@media (max-width: 599px) {
	.score {
		width: 40px;
		margin: 0 5px;
	}
}

.datepicker {
  display: flex;
  justify-content: center;
  margin: 15px 0;
}

.sendButton {
  margin-top: 20px;
}

.errorMessage {
  display: flex;
  justify-content: center;
  color: red;
  margin:auto;
}

.overall-pool {
  display: flex;
  margin: 10px auto;
  justify-content: center;
  font-weight: 400;
  font-size: 1.5rem;
}
</style>
