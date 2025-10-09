<script setup lang="ts">

import { ref } from 'vue'
import PouchDB from 'pouchdb'
//DATA VARIABLES
const counter = ref(0);
const db = new PouchDB('http://admin:admin@localhost:5984/test_infradon2');
//FUNCTIONS / METHODS -> CONTROLLER
const increment = () => {
  counter.value++
}

const initDatabase = () => {

}

const mounted = () => {
    this.initDatabase()
    this.fetchData()
}

</script>

<template>
  <h1>Welcome</h1>
  <p>Counter: {{ counter }}</p>
  <button @click="increment">Increment</button>
</template>
