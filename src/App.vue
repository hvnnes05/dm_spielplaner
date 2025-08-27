<template>
  <div class="p-6 max-w-5xl mx-auto flex justify-center items-center min-h-screen">
    <!-- Spieler hinzufügen -->
    <div v-if="phase === 'start'" class="space-y-2 w-full inline-flex flex-col items-center">
      <img src="/Logo.svg" alt="" class="h-72 mx-auto mb-8" />
      <div class="flex flex-col gap-4 w-full">
        <h2 class="">Name</h2>
        <input v-model="newPlayer" @keyup.enter="addPlayer" placeholder="Spielername" class="border-2 p-2 flex-1 outline-none" />
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
        <input type="file" id="fileUpload" accept="application/json" @change="handleFileUpload" class="hidden" />
        <label for="fileUpload" class="mt-2 py-2 px-4 border-2">Datei hochladen</label>
      </div>

      <button
        :class="['mt-4 px-4 py-2 text-2xl ', players.length >= 8 ? 'bg-gray-900 text-white cursor-pointer' : 'bg-gray-600 text-white cursor-not-allowed']"
        :disabled="players.length < 8"
        @click="startGroups"
      >
        Spiel starten
      </button>
    </div>

    <!-- Gruppenphase -->
    <div v-else-if="phase === 'groups'" class="w-full">
      <div class="grid grid-cols-2 gap-6">
        <div v-for="(group, gIndex) in groups" :key="gIndex" class="p-4 border-2">
          <h3 class="font-semibold mb-2">Gruppe {{ gIndex + 1 }}</h3>
          <div class="grid grid-cols-3 gap-8">
          <div class="col-span-2 divide-y-2">
          <div
            v-for="(match, mIndex) in group.matches"
            :key="mIndex"
            class="flex justify-between items-center py-1"
          >
            <span class="grid w-full" style="grid-template-columns: 40% 20% 40%;">
               <span
                class="cursor-pointer"
                :class="{
                  'text-amber-300': match.winner === match[0],
                }"
                @click="setWinner(gIndex, mIndex, match[0], true, $event)"
              >
                {{ match[0] }}
              </span>
              <span :class="['text-center text-sm', match.tiebreaker ? 'text-red-500' : 'text-gray-500']">vs</span>
              <span
                class="text-right cursor-pointer"
                :class="{
                  'text-amber-300': match.winner === match[1],
                }"
                ref="winnerBtn"
                @click="setWinner(gIndex, mIndex, match[1], true, $event)"
              >
                {{ match[1] }}
              </span>
            </span>
          </div>
          </div>
          <div class="mt-2 text-sm col-span-1 flex flex-col gap-2">
            <span
              v-for="([name, wins], idx) in getSortedWins(group)"
              :key="name"
              :class="getRankStyle(idx)"
              class="p-2"
            >
            <span class="flex">
                <span class="w-6 text-center pr-2">
                {{ getRankEmoji(idx) }} 
                </span>
                <span>
                    {{ name }}: {{ wins }}
                </span>
            </span>
            <span v-if="idx < getSortedWins(group).length - 1"> </span>
            </span>
          </div>
        </div>
        </div>
      </div>

      <div class="flex gap-4 mt-6">
        <button
          v-if="allMatchesDone"
          @click="showWinners"
          class="px-4 py-2 bg-gray-900 text-white"
        >
          Sieger anzeigen
        </button>
        <button
          @click="backToStart"
          class="px-4 py-2 bg-gray-500 text-white"
        >
          Zurück
        </button>
      </div>
    </div>

    <!-- Gewinner -->
    <div v-else-if="phase === 'end'" class="w-full flex flex-col items-center gap-8">
      <h1>Finale</h1>
      <ul class="flex gap-4">
        <li class="py-4 px-8 bg-slate-100 flex flex-col items-center gap-1" v-for="(winner, index) in winners" :key="index">
          <span>Gruppe {{ index + 1 }}</span>
          <h3>{{ winner }}</h3>
        </li>
      </ul>
      <button
        @click="backToGroups"
        class="px-4 py-2 bg-gray-500 text-white"
      >
        Zurück
      </button>
    </div>
  </div>
</template>
<script setup>
import { reactive, ref, computed } from 'vue'
import confetti from 'canvas-confetti'

const phase = ref('start')
const players = reactive([])
const newPlayer = ref('')
const groups = reactive([])
const winners = ref([])
const winnerBtn = ref(null) 

// Spieler-Verwaltung
function addPlayer() {
  if (newPlayer.value.trim()) {
    players.push(newPlayer.value.trim())
    newPlayer.value = ''
  }
}
function removePlayer(index) {
  players.splice(index, 1)
}

// Import aus JSON-Datei
function handleFileUpload(event) {
  const file = event.target.files[0]
  if (!file) return

  const reader = new FileReader()
  reader.onload = (e) => {
    try {
      const parsed = JSON.parse(e.target.result)
      if (Array.isArray(parsed)) {
        players.splice(0, players.length, ...parsed.map(p => String(p).trim()).filter(Boolean))
      } else {
        alert('Die Datei muss ein JSON-Array sein, z. B. ["Alice","Bob"]')
      }
    } catch (err) {
      alert('Ungültige JSON-Datei')
    }
  }
  reader.readAsText(file)
}

// Shuffle
function shuffle(array) {
  return array.sort(() => Math.random() - 0.5)
}

// Matches
function generateMatches(groupPlayers) {
  const matches = []
  for (let i = 0; i < groupPlayers.length; i++) {
    for (let j = i + 1; j < groupPlayers.length; j++) {
      matches.push([groupPlayers[i], groupPlayers[j]])
    }
  }
  for (let i = matches.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1))
    ;[matches[i], matches[j]] = [matches[j], matches[i]]
  }
  return matches
}

// Gruppen starten
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

// Match-Ergebnis
function setWinner(groupIndex, matchIndex, winner, allowChange = false, event = null) {
  const group = groups[groupIndex]
  const match = group.matches[matchIndex]

  if (!match.winner) {
    match.winner = winner
    group.wins[winner]++
  } else if (allowChange && match.winner !== winner) {
    group.wins[match.winner]--
    match.winner = winner
    group.wins[winner]++
  }
  let x = 0.5, y = 0.5
  if (event && event.target) {
    const rect = event.target.getBoundingClientRect()
    x = (rect.left + rect.width / 2) / window.innerWidth
    y = (rect.top + rect.height / 2) / window.innerHeight
  }
  confetti({
    particleCount: 20,
    spread: 180,
    startVelocity: 10,
    gravity: 0.2,
    ticks: 100,
    origin: { 
      x: x,
      y: y
     },
    colors: ['#ead88b']
  })

  // Stechen erzeugen, wenn notwendig
  if (group.matches.every(m => m.winner)) {
    const sorted = getSortedWins(group)
    if (sorted.length > 1 && sorted[0][1] === sorted[1][1]) {
      const [p1, p2] = [sorted[0][0], sorted[1][0]]
      if (!group.matches.some(m => m.tiebreaker)) {
        group.matches.push({ 0: p1, 1: p2, tiebreaker: true })
      }
    }
  }
}

// Sind alle Matches fertig?
const allMatchesDone = computed(() => {
  return groups.every((g) => g.matches.every((m) => m.winner))
})

// Ranking sortieren
function getSortedWins(group) {
  return Object.entries(group.wins)
    .sort((a, b) => {
      if (b[1] !== a[1]) return b[1] - a[1]
      return a[0].localeCompare(b[0], undefined, { sensitivity: 'base' })
    })
}

// Tailwind-Klassen & Emojis je Platz
function getRankStyle(idx) {
  if (idx === 0) return 'bg-yellow-100 font-bold'
  if (idx === 1) return 'bg-gray-100 '
  if (idx === 2) return 'bg-orange-200 '
  return 'border-1 border-gray-300'
}

function getRankEmoji(idx) {
  if (idx === 0) return '🥇'
  if (idx === 1) return '🥈'
  if (idx === 2) return '🥉'
  return idx + 1
}

// Gewinner
function showWinners() {
  confetti({
    particleCount: 100,
    spread: 100,
    scalar: 1.2,
    origin: { y: 0.6 },
    colors: ['#ead88b']
  })
  winners.value = groups.map((g) => getSortedWins(g)[0]?.[0] ?? '')
  phase.value = 'end'
}

// Zurück-Funktion
function backToStart() {
  groups.splice(0, groups.length)
  winners.value = []
  phase.value = 'start'
}
function backToGroups() {
  winners.value = []
  phase.value = 'groups'
}
</script>