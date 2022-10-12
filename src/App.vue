<script setup>
  import { ref } from 'vue'
  import { Skill } from './classes/Skill'

  let hero = ref();
  let villain = ref();
  let announcer = ref();
  let heroTurn = ref(false)
  let battleLog = ref([]);
  let battleEnded = ref(false)
  let usedResilience = ref(false);
  let turnsSinceResilience = ref(0);

  function random(min, max) {
    return Math.floor(Math.random() * (max - min + 1) ) + min;
  }

  const init = async () => {
    await fetch("http://127.0.0.1:3000/init");

    hero.value = await fetch("http://127.0.0.1:3000/hero", {
      method: 'get',
    }).then(response => response.json());

    villain.value = await fetch("http://127.0.0.1:3000/villain", {
      method: 'get',
    }).then(response => response.json());

    announcer.value = "Battle has begun!";
    battleLog.value = [];
    battleEnded.value = false;
    usedResilience.value = false;
    turnsSinceResilience.value = 0;

    if(hero.value.speed !== villain.value.speed) {
      heroTurn.value = hero.value.speed > villain.value.speed
    }
    else heroTurn.value = hero.value.luck > villain.value.luck
  }

  const battle = (attacker, defender) => {

    if(battleLog.value.length === 20) {
      announcer.value = "Fighters got tired and left. Game over. "
      battleEnded.value = true;
      return;
    }

    let damage = attacker.strength - defender.defence;

    const roll = random(1, 100);
    
    let log = '';

    if(heroTurn.value) {
      if(roll === 1) {
        const criticalStrike = new Skill(3);
        damage *= criticalStrike.damageMultiplier;
        log += 'CRITICAL STRIKE!! Triple damage!';
      }
      else if(roll <= 10) {
        const criticalStrike = new Skill(2);
        damage *= criticalStrike.damageMultiplier;
        log += 'CRITICAL STRIKE! Double damage!'
      }

      if(usedResilience.value) {
        turnsSinceResilience.value++
      }
    }
    else {
      if(!usedResilience.value || turnsSinceResilience.value === 2) {
        if(roll <= 20) {
          const resilience = new Skill(0.5);
          damage *= resilience.damageMultiplier;
          log += 'RESILIENCE! Damage is halved!'
          usedResilience.value = !usedResilience.value
          turnsSinceResilience.value = 0
        }
      }
    }

    defender.health -= damage;
    
    if(defender.health <= 0) {
      if(defender.is_hero) { 
        battleLog.value.push(`${log} Villain attacks Hero for ${damage} damage! Hero has fallen...`);
        announcer.value = 'Battle ended. Hero has fallen...';
      }
      else {
        battleLog.value.push(`${log} Hero attacks Villain for ${damage} damage! Villain is slain!!`);
        announcer.value = 'Battle ended. Villain is slain!';
      }
      battleEnded.value = true;
      return;
    }

    battleLog.value.push(`${log} ${heroTurn.value ? 'Hero' : 'Villain'} attacks ${heroTurn.value ? 'Villain' : 'Hero'} for ${damage} damage!`);
    heroTurn.value = !heroTurn.value;
  }

</script>

<template>
  <div class="flex flex-col justify-center items-center my-8">
    <span v-if="announcer === undefined" class="italic text-4xl">Once upon a time there was a bright hero venturing in the Terminal Valley...</span>
    <span @click="init" class="px-16 py-8 bg-red-500 font-bold rounded-lg text-white text-4xl my-4 cursor-pointer">
      {{announcer === undefined ? 'START' : 'RESTART' }}
    </span>
    <span class="text-3xl">{{ announcer }}</span>

    <div class="flex flex-row space-x-16 m-4">

      <div class="flex flex-col justify-center items-center" v-if="announcer !== undefined">
        <span class="underline m-1">HERO</span>
        <span v-for="stat, index in Object.keys(hero)" :key="index">
          <div class="row items-center justify-center space-x-8" v-if="stat !== 'id' && stat !== 'is_hero'">
            <span class="font-bold">{{ stat }}</span>
            <span class="italic" >{{ hero[stat] }}</span>
          </div>
        </span>
      </div>

      <div class="flex flex-col justify-center items-center" v-if="announcer !== undefined">
        <span class="underline m-1">VILLAIN</span>
        <span v-for="stat, index in Object.keys(villain)" :key="index">
          <div class="row items-center justify-center space-x-8" v-if="stat !== 'id' && stat !== 'is_hero'">
            <span class="font-bold">{{ stat }}</span>
            <span class="italic" >{{ villain[stat] }}</span>
          </div>
        </span>
      </div>
    </div>

    <div v-if="announcer !== undefined" class="m-4">
      <button v-if="!battleEnded" class="px-8 py-4 bg-red-500 text-3xl cursor-pointer rounded-lg" @click="heroTurn ? battle(hero, villain) : battle(villain, hero)">FIGHT!</button>
    </div>
    
    <div v-if="announcer !== undefined" class="flex flex-col justify-center items-center">
      <span class="font-bold">Battle log</span>
      <span v-for="(round, index) in battleLog" class="space-y-2">
        <span class="italic">Round {{ index + 1 }}:</span>
        {{ round }}
      </span>
    </div>
  </div>
</template>
