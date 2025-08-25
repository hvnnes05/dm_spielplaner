<template>
  <div class="p-6 max-w-5xl mx-auto">
    <!-- Spieler hinzufügen -->
    <div v-if="phase === 'start'" class="space-y-4">
      <h1 class="text-2xl font-bold">Spieler eingeben</h1>

      <div class="flex space-x-2">
        <input v-model="newPlayer" placeholder="Spielername" class="border p-2 rounded flex-1" />
        <button @click="addPlayer" class="px-4 py-2 bg-green-500 text-white rounded">Hinzufügen</button>
      </div>

      <ul class="list-disc ml-6 mt-2">
        <li v-for="(player, index) in players" :key="index">{{ player }}</li>
      </ul>

      <div class="mt-4">
        <h2 class="text-lg font-semibold">Optional: Spieler per JSON hochladen</h2>
        <input type="file" accept="application/json" @change="handleFileUpload" class="mt-2" />
      </div>

      <button v-if="players.length >= 4" @click="startGroups" class="mt-4 px-4 py-2 bg-purple-600 text-white rounded">Gruppenphase starten</button>
    </div>

    <!-- Gruppenphase -->
    <div v-else-if="phase === 'groups'">
      <h2 class="text-xl font-bold mb-4">Gruppenphase</h2>
      <div class="grid grid-cols-2 gap-6">
        <div v-for="(group, gIndex) in groups" :key="gIndex" class="p-4 border rounded shadow">
          <h3 class="font-semibold mb-2">Gruppe {{ gIndex + 1 }}</h3>
          <div
            v-for="(match, mIndex) in group.matches"
            :key="mIndex"
            class="flex justify-between items-center border-b py-1"
            :class="{
              'bg-green-100': match.winner,
            }"
          >
            <span>
              <span :class="{ 'font-bold text-green-700': match.winner === match[0] }">{{ match[0] }}</span>
              vs
              <span :class="{ 'font-bold text-green-700': match.winner === match[1] }">{{ match[1] }}</span>
            </span>
            <div class="space-x-2" v-if="!match.winner">
              <button @click="setWinner(gIndex, mIndex, match[0])" class="px-2 py-1 bg-green-500 text-white rounded">{{ match[0] }}</button>
              <button @click="setWinner(gIndex, mIndex, match[1])" class="px-2 py-1 bg-green-500 text-white rounded">{{ match[1] }}</button>
            </div>
            <div v-else class="text-sm text-green-700">✔</div>
          </div>
          <div class="mt-2 text-sm">Siege: {{ JSON.stringify(group.wins) }}</div>
        </div>
      </div>

      <button v-if="allMatchesDone" @click="showWinners" class="mt-6 px-4 py-2 bg-blue-600 text-white rounded">Sieger anzeigen</button>
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

// Matches generieren & leicht mischen, sodass Spieler nicht direkt mehrfach hintereinander spielen
function generateMatches(groupPlayers) {
  let matches = []
  for (let i = 0; i < groupPlayers.length; i++) {
    for (let j = i + 1; j < groupPlayers.length; j++) {
      matches.push([groupPlayers[i], groupPlayers[j]])
    }
  }
  // Fisher-Yates shuffle
  for (let i = matches.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[matches[i], matches[j]] = [matches[j], matches[i]]
  }
  // einfache Heuristik: nachträglich sortieren, damit gleiche Spieler nicht so oft hintereinander kommen
  matches.sort((a, b) => {
    const aPlayers = a.join('')
    const bPlayers = b.join('')
    return aPlayers.localeCompare(bPlayers)
  })
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
      wins: Object.fromEntries(groupPlayers.map(p => [p, 0]))
    })
  }
  phase.value = 'groups'
}

function setWinner(groupIndex, matchIndex, winner) {
  const group = groups[groupIndex]
  if (!group.matches[matchIndex].winner) {
    group.matches[matchIndex].winner = winner
    group.wins[winner]++
  }
}

const allMatchesDone = computed(() => {
  return groups.every(g => g.matches.every(m => m.winner))
})

function showWinners() {
  winners.value = groups.map(g => Object.entries(g.wins).sort((a,b) => b[1]-a[1])[0][0])
  phase.value = 'end'
}
</script>
