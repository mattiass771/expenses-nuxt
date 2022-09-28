<template>
  <h3>Aktuálny projekt:</h3>
  <h1>{{sessionName}} </h1>
  <div><button @click="removeSession">Vymazať projekt</button></div>

  <div class="controls">
    <div class="create">
      <h3>Vytvor nový projekt:</h3>
      <button @click="createNewSession">
        Začať
      </button>
    </div>
    <div class="choose">
      <h3>Výber projektov:</h3>
      <select @change="changeSession($event)">
        <option :selected="true"></option>
        <option v-for="session in allSessions" v-bind:key="session">
          {{session}}
        </option>
      </select>
    </div>
  </div>

  <div class="inputs">
    <div class="input-group">
      <label
        for="name-input"
      >
        Názov položky
      </label>
      <br>
      <input
        id="name-input"
        type="text"
        v-model="name"
        @keydown.enter="handleAddItem"
      >
    </div>
    <div class="input-group">
      <label
        for="plannedPrice-input"
      >
        Plánovaná cena
      </label>
      <br>
      <input
        id="plannedPrice-input"
        type="number"
        v-model="plannedPrice"
        @keydown.enter="handleAddItem"
      >
    </div>
    <div class="input-group">
      <label
        for="plannedPrice-input"
      >
        Účtovaná cena
      </label>
      <br>
      <input
        id="plannedPrice-input"
        type="number"
        v-model="billedPrice"
        @keydown.enter="handleAddItem"
      >
    </div>
  </div>
  <button class="submit" v-on:click="handleAddItem">
    Pridať
  </button>

  <p>
    <em>Kliknutím na názov riadku možno pričítavať a odčítavať použitím zápornej hodnoty. Ak obe hodnoty nastavíš na negatívne, vymažeš tým celý riadok.</em>
  </p>

  <table class="tabulka">
    <thead>
      <tr>
        <th>Názov položky</th>
        <th>Plánovaná cena</th>
        <th>Účtovaná cena</th>
        <th>Rozdiel</th>
      </tr>
    </thead>
    <tbody>
      <tr v-for="item in items" v-bind:key="item.nameComputed">
        <td class="clickable-cell" @click="name = item.name">{{item.name}}</td>
        <td>{{item.plannedPrice}} €</td>
        <td>{{item.billedPrice}} €</td>
        <td v-bind:class="Number(item.plannedPrice) - Number(item.billedPrice) >= 0 ? 'bilance-positive' : 'bilance-negative'">
          {{item.plannedPrice - item.billedPrice}} €
        </td>
      </tr>
      <tr>
        <td>Sumár</td>
        <td>{{plannedTotal}} €</td>
        <td>{{billedTotal}} €</td>
        <td v-bind:class="diffTotal >= 0 ? 'bilance-positive' : 'bilance-negative'">
          {{diffTotal}} €
        </td>
      </tr>
    </tbody>
  </table>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { uniqueNamesGenerator, adjectives, colors, animals } from 'unique-names-generator'

interface Item {
  name: string
  nameComputed: string
  plannedPrice: number
  billedPrice: number
}

const defaultItem: Item = {
  name: '',
  nameComputed: '',
  plannedPrice: 0,
  billedPrice: 0,
}

const name = ref(defaultItem.name)
const plannedPrice = ref(defaultItem.plannedPrice)
const billedPrice = ref(defaultItem.billedPrice)

const items = ref<Item[]>()
const sessionName = ref<string>('')
const allSessions = ref<string[]>()

onMounted(() => {
  const loadLastSession = localStorage.getItem('estim-val-price-diff-app-last')
  items.value = JSON.parse(localStorage.getItem(`estim-val-price-diff-app-${loadLastSession}`)!) ?? []
  const shortName: string = uniqueNamesGenerator({
    dictionaries: [colors, adjectives, animals]
  });
  sessionName.value = loadLastSession && loadLastSession.length > 0 ? loadLastSession : shortName
  allSessions.value = Object.keys(localStorage).filter(val => val.startsWith('estim-val-price-diff-app') && val !== 'estim-val-price-diff-app-last').map(val => val.replace('estim-val-price-diff-app-', ''))
})

const changeSession = (event: any) => {
  const newSessionName = `estim-val-price-diff-app-${event.target.value}`
  items.value = JSON.parse(localStorage.getItem(newSessionName)!)
  sessionName.value = event.target.value
}

const removeSession = () => {
  localStorage.removeItem(`estim-val-price-diff-app-${sessionName.value}`)
  allSessions.value = allSessions.value?.filter(val => val !== sessionName.value)
  localStorage.setItem('estim-val-price-diff-app-last', allSessions?.value?.[allSessions?.value.length-1] ?? '')
  window.location.reload()
}

const createNewSession = () => {
  const newSessionName: string = uniqueNamesGenerator({
    dictionaries: [colors, adjectives, animals]
  });
  items.value = []
  sessionName.value = newSessionName
  const newSessions = Object.keys(localStorage).filter(val => val.startsWith('estim-val-price-diff-app') && val !== 'estim-val-price-diff-app-last').map(val => val.replace('estim-val-price-diff-app-', ''))
  allSessions.value = [...newSessions, newSessionName]
}

const billedTotal = computed(() => items.value ? items.value!.reduce<number>((prev, curr) => {
    return prev + Number(curr.billedPrice)
  }, 0) : 0
);
const plannedTotal = computed(() => items.value ? items.value!.reduce<number>((prev, curr) => {
    return prev + Number(curr.plannedPrice)
  }, 0) : 0
);
const diffTotal = computed(() => items.value ? items.value!.reduce<number>((prev, curr) => {
    return prev + (Number(curr.plannedPrice) - Number(curr.billedPrice))
  }, 0) : 0
);

const handleAddItem = () => {
  if (name.value.length > 0) {
    const nameComputed = name.value.toLowerCase().replaceAll(' ', '_')
    const ix = items.value!.findIndex(val => val.nameComputed === nameComputed)
    if (ix >= 0 && items.value) {
      const newBilledPrice = items.value[ix].billedPrice + billedPrice.value
      const newPlannedPrice = items.value[ix].plannedPrice + plannedPrice.value
      if (newBilledPrice <= 0 && newPlannedPrice <= 0) {
        items.value.splice(ix, 1)
      } else {
        items.value[ix] = {
          name: items.value[ix].name,
          nameComputed,
          billedPrice: newBilledPrice,
          plannedPrice: newPlannedPrice,
        }
      }
    } else {
      items.value?.push({
        name: name.value, nameComputed, plannedPrice: plannedPrice.value, billedPrice: billedPrice.value
      })
    }
    localStorage.setItem(`estim-val-price-diff-app-${sessionName.value}`, JSON.stringify(items.value))
    localStorage.setItem('estim-val-price-diff-app-last', sessionName.value)
    if (!allSessions.value?.includes(sessionName.value)) {
      allSessions.value?.push(sessionName.value)
    }
    name.value = defaultItem.name
    plannedPrice.value = defaultItem.plannedPrice
    billedPrice.value = defaultItem.billedPrice
  }
}

</script>

<style scoped>
  em {
    font-size: 0.8rem;
  }
  h1 {
    background-color: rgba(99, 107, 255, 0.25);
    padding: 1rem 0rem;
    letter-spacing: 0.15rem;
    border-radius: 2px;
    border: solid 2px #646cff;
  }
  .controls {
    display: grid;
    grid-template-columns: 1fr 1fr;
    margin-bottom: 2rem;
  }

  input {
    min-width: 80%;
  }

  .submit {
    margin-top: 1rem;
  }

  .inputs {
    display: grid;
    grid-template-columns: 1fr 1fr 1fr;
  }
  .tabulka {
    width: 100%;
    margin-top: 1rem;
  }

  .choose {
    margin-bottom: 1rem;
  }

  .clickable-cell {
    box-shadow: inset 0px 0px 10px -3px rgb(160, 205, 213);
  }

  .clickable-cell:hover {
    cursor: pointer;
    box-shadow: 2px 2px 10px -2px rgb(178, 207, 226);
  }

  table, th, td {
    border: 1px solid #ffffffde;
    padding: 5px;
  }

  table {
    border-radius: 5px;
  }
  .bilance-positive {
    background-color: rgb(99, 158, 99, 0.5);
  }
  .bilance-negative {
    background-color: rgb(180, 75, 75, 0.5);
  }

  @media screen and (max-width: 400px) {
    table, td, th {
      font-size: 80%;
      padding: 0.2rem;
    }
    h1 {
      font-size: 0.9rem !important;
    }
    em {
      font-size: 0.6rem !important;
    }
  }

  @media screen and (max-width: 768px) {
    em {
      font-size: 0.7rem !important;
    }
    
    h1 {
      font-size: 1.15rem;
      padding: 0.35rem 0rem;
    }

    h3 {
      font-size: 1rem;
    }
    .inputs {
      display: grid;
      grid-template-columns: 1fr;
    }

    .controls {
      grid-template-columns: 1fr;
    }
  }
</style>
