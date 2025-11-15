<script setup>
import { ref, onMounted } from 'vue'

// ===== Config =====
const API_BASE = import.meta.env.VITE_API_BASE ?? 'http://127.0.0.1:8000/api/v1'

// ===== Ejercicio 1: Pokémon =====
const pokeName = ref('')
const pokeResult = ref(null)
const pokeError = ref('')
const pokeLoading = ref(false)

async function fetchPokemon () {
  pokeError.value = ''
  pokeResult.value = null
  const name = pokeName.value.trim().toLowerCase()
  if (!name) { pokeError.value = 'Enter a Pokémon name.'; return }

  try {
    pokeLoading.value = true
    const res = await fetch(`https://pokeapi.co/api/v2/pokemon/${encodeURIComponent(name)}`)
    if (!res.ok) throw new Error('Not found')
    const data = await res.json()
    pokeResult.value = {
      name: data.name,
      img: data.sprites?.other?.['official-artwork']?.front_default || data.sprites?.front_default
    }
  } catch {
    pokeError.value = 'Pokémon not found 😿'
  } finally {
    pokeLoading.value = false
  }
}

// ===== Ejercicio 2: Update Robotics Kit Name (Laravel API) =====
const kits = ref([])
const selectedKitId = ref('')
const newKitName = ref('')
const kitMsg = ref('')
const kitError = ref('')
const kitLoading = ref(false)

async function loadKits () {
  kits.value = []
  kitError.value = ''
  try {
    const res = await fetch(`${API_BASE}/kits?per_page=100`)
    if (!res.ok) throw new Error('Failed to load kits')
    const payload = await res.json()
    // Si tu API devuelve paginator: payload.data.data; si devuelve array directo: payload.data
    const rows = Array.isArray(payload.data?.data) ? payload.data.data : (payload.data ?? [])
    kits.value = rows
    if (rows.length && !selectedKitId.value) selectedKitId.value = rows[0].id
  } catch {
    kitError.value = 'Could not load robotics kits.'
  }
}

async function updateKitName () {
  kitMsg.value = ''
  kitError.value = ''
  if (!selectedKitId.value || !newKitName.value.trim()) {
    kitError.value = 'Select a kit and enter a new name.'
    return
  }
  try {
    kitLoading.value = true
    // Para PUT, reenviamos los demás campos requeridos por el validador (sku/price existentes)
    const current = kits.value.find(k => k.id === selectedKitId.value) || {}
    const body = {
      name: newKitName.value.trim(),
      sku: current.sku ?? '',
      price: current.price ?? 0,
      description: current.description ?? null
    }

    const res = await fetch(`${API_BASE}/kits/${selectedKitId.value}`, {
      method: 'PUT', // usa 'PATCH' si tu controlador admite actualización parcial
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body)
    })
    if (!res.ok) {
      const err = await res.json().catch(() => ({}))
      throw new Error(err.error || 'Update failed')
    }
    kitMsg.value = 'Kit name updated ✅'
    await loadKits()
    newKitName.value = ''
  } catch (e) {
    kitError.value = e.message || 'Could not update kit.'
  } finally {
    kitLoading.value = false
  }
}

onMounted(loadKits)
</script>

<template>
  <main class="min-h-screen bg-slate-50 text-slate-900 p-6">
    <div class="max-w-3xl mx-auto grid gap-10">

      <!-- Ejercicio 1: Pokémon -->
      <section class="bg-white rounded-2xl shadow border p-6">
        <h2 class="text-xl font-semibold mb-4">Pokémon Finder (Fetch API)</h2>
        <div class="flex gap-3 mb-4">
          <input
            v-model="pokeName"
            type="text"
            placeholder="e.g. pikachu"
            class="flex-1 border rounded-md px-3 py-2"
            @keyup.enter="fetchPokemon"
          />
          <button
            class="px-4 py-2 rounded-md bg-blue-600 text-white disabled:opacity-60"
            :disabled="pokeLoading"
            @click="fetchPokemon"
          >
            {{ pokeLoading ? 'Loading…' : 'Search' }}
          </button>
        </div>

        <p v-if="pokeError" class="text-red-600 mb-3">{{ pokeError }}</p>

        <div v-if="pokeResult" class="flex items-center gap-4">
          <img v-if="pokeResult.img" :src="pokeResult.img" :alt="pokeResult.name" class="h-28 w-28 object-contain" />
          <div class="text-lg capitalize font-medium">{{ pokeResult.name }}</div>
        </div>
      </section>

      <!-- Ejercicio 2: Update Robotics Kit Name -->
      <section class="bg-white rounded-2xl shadow border p-6">
        <h2 class="text-xl font-semibold mb-4">Update Robotics Kit Name (Fetch → Laravel API)</h2>

        <div class="grid sm:grid-cols-2 gap-4 mb-4">
          <div>
            <label class="block text-sm font-medium mb-1">Robotics Kit</label>
            <select v-model="selectedKitId" class="w-full border rounded-md px-3 py-2">
              <option v-for="k in kits" :key="k.id" :value="k.id">
                #{{ k.id }} — {{ k.name }} ({{ k.sku }})
              </option>
            </select>
          </div>
          <div>
            <label class="block text-sm font-medium mb-1">New name</label>
            <input v-model="newKitName" type="text" class="w-full border rounded-md px-3 py-2" placeholder="e.g. AI Rover v2" />
          </div>
        </div>

        <div class="flex gap-3">
          <button
            class="px-4 py-2 rounded-md bg-emerald-600 text-white disabled:opacity-60"
            :disabled="kitLoading"
            @click="updateKitName"
          >
            {{ kitLoading ? 'Updating…' : 'Update Kit' }}
          </button>
          <button class="px-4 py-2 rounded-md border" @click="loadKits">Reload</button>
        </div>

        <p v-if="kitMsg" class="text-emerald-600 mt-3">{{ kitMsg }}</p>
        <p v-if="kitError" class="text-red-600 mt-3">{{ kitError }}</p>
      </section>

    </div>
  </main>
</template>

<style>
html,body,#app{ height:100%; }
</style>
