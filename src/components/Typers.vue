<template>
    <div class="table-container">
        <ui-table class="table" :data="typers" :thead="head" :tbody="body" fullwidth sortIconAlignEnd>
	        <template #country="{data}">
		        <img class="flag" :src="getCountry(data.country)" :aria-describedby="data.country"/>
		        <ui-tooltip :id="data.country">{{data.country}}</ui-tooltip>
	        </template>
        </ui-table>
    </div>
</template>

<script>
import BalmUI from 'balm-ui'
import axios from 'axios'
import authHeader from './../service/auth-header'

export default {
    name: 'Typers',
    data() {
        return {
            typers: [],
            head: [{
                value: 'Użytkownik',
                align: 'center'
            }, {
                value: 'Trafione',
	            sort: 'desc',
	            columnId: 'correctTypings',
                align: 'center'
            }, {
				value: 'Saldo',
	            sort: 'none',
	            columnId: 'balance',
	            align: 'center'
            }, {
				value: 'Kraj',
	            align: 'center'
            }],
            body: [{
                field: 'username',
                align: 'center'
            }, {
                field: 'correctTypings',
                align: 'center'
            }, {
				field: 'balance',
	            align: 'center'
            },{
				slot: 'country',
	            align: 'center'
            }]
        }
    },
    methods: {
        getTypers() {
            axios.get(this.$store.state.origin + ':8080/typings/typerScores', {
                headers: authHeader()
            })
            .then((response) => {
                this.typers = response.data
            })
        },
	    getCountry(countryString) {
		    if(countryString){
			    return new URL(`../assets/icons/${countryString}.svg.webp`, import.meta.url).href
		    }
	    }
    },
    mounted() {
        this.getTypers()
    }
}

</script>

<style scoped>
.table-container {
    display: flex;
    width: 100%;
    justify-content: center;
    margin: 20px auto;
    max-width: 500px;
}
.table {
    width: 100%;
}
.flag {
	width: 60px;
	height: 40px;
	border: 1px solid black;
	display: flex;
	margin: auto;
}
</style>