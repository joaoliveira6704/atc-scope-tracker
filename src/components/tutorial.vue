<script setup>
import { reactive } from "vue";
const myMessage = "Teste";
const num = 6;
const add = (x, y) => x + y;

const youtube = "https://www.youtube.com";

let initState = reactive({ count: 0 });
let secState = reactive({ val: { count: 0 }, user: ["admin", "john"] });
</script>

<template>
  <h1>{{ myMessage }}</h1>
  <h1>{{ num }}</h1>
  <h1>{{ add(num, 1) }}</h1>
  <a :href="youtube">Youtube!</a>
  <h1>count: {{ initState.count }}</h1>
  <button @click="initState.count++">Aumentar</button>
  <button @click="initState.count--">Diminuir</button>
  <hr />
  <h1>Current sec count: {{ secState.val.count }}</h1>
  <h1>Current user: {{ secState.user }}</h1>
</template>

<style scoped></style>
