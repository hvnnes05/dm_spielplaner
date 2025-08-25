<template>
  <div class="p-6 max-w-5xl mx-auto flex justify-center items-center min-h-screen">
    <!-- Spieler hinzufügen -->
    <div v-if="phase === 'start'" class="space-y-2 w-full flex-inline flex-col justify-center">
      <img src="../Logo.svg" alt="" class="h-72 mx-auto mb-8" />
      <h1 class="">Name</h1>
      <div class="flex">
        <input v-model="newPlayer" @keyup.enter="addPlayer" placeholder="Spielername" class="border-2 p-2 flex-1" />
        <button @click="addPlayer" class="px-4 py-2 bg-gray-900 text-white">Hinzufügen</button>
      </div>

      <ul class="flex gap-2 mt-2 flex-wrap">
        <li
          v-for="(player, index) in players"
          :key="index"
          class="py-2 px-4 bg-slate-100 flex items-center gap-2 cursor-pointer hover:bg-red-200"
          @dblclick="removePlayer(index)"
        >
          {{ player }}
        </li>
      </ul>

      <div class="mt-4">
        <input type="file" accept="application/json" @change="handleFileUpload" class="mt-2 py-2 px-4 border-2" />
      </div>

      <button
        :class="['mt-4 px-4 py-2 ', players.length >= 4 ? 'bg-gray-900 text-white cursor-pointer' : 'bg-gray-600 text-white cursor-not-allowed']"
        :disabled="players.length < 4"
        @click="startGroups"
      >
        Gruppenphase starten
      </button>
    </div>

    <!-- Gruppenphase -->
    <div v-else-if="phase === 'groups'">
      <div class="grid grid-cols-2 gap-6">
        <div v-for="(group, gIndex) in groups" :key="gIndex" class="p-4 border-2">
          <h3 class="font-semibold mb-2">Gruppe {{ gIndex + 1 }}</h3>
          <div
            v-for="(match, mIndex) in group.matches"
            :key="mIndex"
            class="flex justify-between items-center border-b py-1"
          >
            <span class="grid w-full" style="grid-template-columns: 40% 20% 40%;">
               <span
                class="cursor-pointer"
                :class="{
                  'text-amber-300': match.winner === match[0],
                }"
                @click="setWinner(gIndex, mIndex, match[0], true)"
              >
                {{ match[1] }}
              </span>
              <span class="text-center text-gray-500 text-sm">vs</span>
              <span
                class="text-right cursor-pointer"
                :class="{
                  'text-amber-300': match.winner === match[1],
                }"
                @click="setWinner(gIndex, mIndex, match[1], true)"
              >
                {{ match[1] }}
              </span>

            </span>
          </div>
          <div class="mt-2 text-sm">Siege: {{ JSON.stringify(group.wins) }}</div>
        </div>
      </div>

      <button
        v-if="allMatchesDone"
        @click="showWinners"
        class="mt-6 px-4 py-2 bg-blue-600 text-white rounded"
      >
        Sieger anzeigen
      </button>
    </div>

    <!-- Gewinner -->
    <div v-else-if="phase === 'end'">
      <h2 class="text-2xl font-bold mb-4">Gruppensieger</h2>
      <ul class="list-disc ml-6">
        <li v-for="(winner, index) in winners" :key="index">Gruppe {{ index + 1 }}: {{ winner }}</li>
      </ul>
    </div>
  </div>
</template>

<script setup>
import { reactive, ref, computed } from 'vue'

const phase = ref('start')
const players = reactive([])
const newPlayer = ref('')
const groups = reactive([])
const winners = ref([])

function addPlayer() {
  if (newPlayer.value.trim()) {
    players.push(newPlayer.value.trim())
    newPlayer.value = ''
  }
}

function removePlayer(index) {
  players.splice(index, 1)
}

function handleFileUpload(event) {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const parsed = JSON.parse(e.target.result)
      if (Array.isArray(parsed)) {
        players.splice(0, players.length, ...parsed)
      }
    } catch (err) {
      alert('Ungültige JSON-Datei')
    }
  }
  reader.readAsText(file)
}

function shuffle(array) {
  return array.sort(() => Math.random() - 0.5)
}

function generateMatches(groupPlayers) {
  let matches = []
  for (let i = 0; i < groupPlayers.length; i++) {
    for (let j = i + 1; j < groupPlayers.length; j++) {
      matches.push([groupPlayers[i], groupPlayers[j]])
    }
  }
  // Shuffle
  for (let i = matches.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[matches[i], matches[j]] = [matches[j], matches[i]]
  }
  return matches
}

function startGroups() {
  const shuffled = shuffle([...players])
  groups.splice(0, groups.length)

  for (let i = 0; i < 4; i++) {
    const groupPlayers = shuffled.filter((_, idx) => idx % 4 === i)
    groups.push({
      players: groupPlayers,
      matches: generateMatches(groupPlayers),
      wins: Object.fromEntries(groupPlayers.map((p) => [p, 0]))
    })
  }
  phase.value = 'groups'
}

function setWinner(groupIndex, matchIndex, winner, allowChange = false) {
  const group = groups[groupIndex]
  const match = group.matches[matchIndex]

  if (!match.winner) {
    match.winner = winner
    group.wins[winner]++
  } else if (allowChange) {
    // rückgängig machen
    group.wins[match.winner]--
    match.winner = winner
    group.wins[winner]++
  }
}

const allMatchesDone = computed(() => {
  return groups.every((g) => g.matches.every((m) => m.winner))
})

function showWinners() {
  winners.value = groups.map((g) =>
    Object.entries(g.wins).sort((a, b) => b[1] - a[1])[0][0]
  )
  phase.value = 'end'
}
</script>
