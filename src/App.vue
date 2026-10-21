<script setup>
import { computed, ref, watch } from "vue"
import hmitImage from "./assets/nemorphism HMIT (2).png"

const defaults = [
  { id: 1, name: "Paslon 01", tagline: "Solid, progresif, berdampak", color: "#cf2638" },
  { id: 2, name: "Paslon 02", tagline: "Kolaborasi untuk himpunan", color: "#1769bd" }
]

const saved = JSON.parse(localStorage.getItem("himpunan-counter") || "null")
const candidates = ref(saved?.candidates?.slice(0, 2) || defaults.map(x => ({ ...x, votes: 0 })))
const invalid = ref(saved?.invalid || 0)
const logs = ref(saved?.logs || [])
const activeTab = ref("live")
const showReset = ref(false)
const showManage = ref(false)
const isFullscreen = ref(false)
const dark = ref(saved?.dark || false)
const draftCandidates = ref([])

const total = computed(() => candidates.value.reduce((sum, c) => sum + c.votes, 0))
const counted = computed(() => total.value + invalid.value)
const ranked = computed(() => [...candidates.value].sort((a, b) => b.votes - a.votes))
const winner = computed(() => ranked.value[0])
const percent = votes => total.value ? ((votes / total.value) * 100).toFixed(1) : "0.0"
const time = () => new Intl.DateTimeFormat("id-ID", { hour: "2-digit", minute: "2-digit", second: "2-digit" }).format(new Date())

function log(label, amount, kind = "sah") {
  logs.value.unshift({ id: Date.now(), label, amount, kind, time: time() })
  logs.value = logs.value.slice(0, 80)
}

function change(candidate, amount) {
  if (amount < 0 && candidate.votes === 0) return
  candidate.votes += amount
  log(candidate.name, amount)
}

function changeOther(amount) {
  if (amount < 0 && invalid.value === 0) return
  invalid.value += amount
  log("Tidak sah", amount, "invalid")
}

function undo() {
  const last = logs.value.shift()
  if (!last) return
  if (last.kind === "sah") {
    const target = candidates.value.find(c => c.name === last.label)
    if (target && target.votes >= last.amount) target.votes -= last.amount
  } else if (invalid.value >= last.amount) {
    invalid.value -= last.amount
  }
}

function resetAll() {
  candidates.value.forEach(c => c.votes = 0)
  invalid.value = 0
  logs.value = []
  showReset.value = false
}

function fullscreen() {
  if (!document.fullscreenElement) document.documentElement.requestFullscreen?.()
  else document.exitFullscreen?.()
}

function syncFullscreen() {
  isFullscreen.value = Boolean(document.fullscreenElement)
}

function openManage() {
  draftCandidates.value = candidates.value.map(candidate => ({ ...candidate }))
  showManage.value = true
}

function addCandidate() {
  const nextId = Math.max(0, ...draftCandidates.value.map(candidate => candidate.id)) + 1
  draftCandidates.value.push({ id: nextId, name: `Paslon ${String(nextId).padStart(2, "0")}`, tagline: "Tambahkan deskripsi", color: nextId % 2 ? "#cf2638" : "#1769bd", votes: 0 })
}

function removeCandidate(index) {
  if (draftCandidates.value.length <= 1) return
  draftCandidates.value.splice(index, 1)
}

function saveCandidates() {
  candidates.value = draftCandidates.value.filter(candidate => candidate.name.trim()).map(candidate => ({ ...candidate, name: candidate.name.trim() }))
  showManage.value = false
}

watch([candidates, invalid, logs, dark], () => {
  localStorage.setItem("himpunan-counter", JSON.stringify({
    candidates: candidates.value,
    invalid: invalid.value,
    logs: logs.value,
    dark: dark.value
  }))
}, { deep: true })

if (typeof document !== "undefined") document.addEventListener("fullscreenchange", syncFullscreen)
</script>

<template>
  <div class="app" :class="{ dark }">
    <header class="topbar">
      <div class="brand">
        <div>
          <strong>HIMPUNAN MAHASISWA INFORMATION TECHNOLOGY (HMIT)</strong>
          <span>Telkom University Jakarta</span>
        </div>
      </div>
      <div class="header-actions">
        <button class="icon-btn" @click="dark = !dark" :title="dark ? 'Tema Terang' : 'Tema Gelap'">
          {{ dark ? "☀" : "◐" }}
        </button>
        <button class="icon-btn" @click="fullscreen" title="Layar Penuh">⛶</button>
        <button class="btn manage-btn" @click="openManage">Kelola Paslon</button>
        <button class="btn danger ghost" @click="showReset = true">Reset Data</button>
      </div>
    </header>

    <main>
      <section class="hero">
        <div>
          <h1>Rekapitulasi Suara<br /><em>Musyawarah HMIT 2026</em></h1>
          <p class="sub">Pemilihan Ketua Himpunan · Periode 2026 / 2027</p>
        </div>
      </section>

      <section class="stats-row">
        <article class="stat neu main-stat">
          <span class="stat-label">Total suara sah</span>
          <strong class="stat-num">{{ total.toLocaleString("id-ID") }}</strong>
          <small>dari {{ counted.toLocaleString("id-ID") }} suara masuk</small>
        </article>
        <article class="stat neu">
          <span class="stat-label">Tidak sah</span>
          <strong class="stat-num sm">{{ invalid }}</strong>
          <div class="mini-actions">
            <button class="mini-btn minus" @click="changeOther(-1)">−</button>
            <button class="mini-btn plus" @click="changeOther(1)">+</button>
          </div>
        </article>
        <img class="hmit-inline-art" :src="hmitImage" alt="HMIT neumorphism identity artwork" />
      </section>

      <nav class="tabs">
        <button :class="{ active: activeTab === 'live' }" @click="activeTab = 'live'">Live Count</button>
        <button :class="{ active: activeTab === 'log' }" @click="activeTab = 'log'">
          Riwayat Log <b>{{ logs.length }}</b>
        </button>
      </nav>

      <section v-if="activeTab === 'live'" class="content-grid">
        <div class="candidates-col">
          <div class="section-head">
            <div>
              <p class="eyebrow">DAFTAR PESERTA</p>
              <h2>Perolehan Suara</h2>
            </div>
            <button class="btn undo-btn" :disabled="!logs.length" @click="undo">↶ Undo</button>
          </div>

          <article
            v-for="(candidate, index) in candidates"
            :key="candidate.id"
            class="candidate-card neu"
            :class="{ winner: candidate.id === winner?.id && total > 0 }"
          >
            <div class="rank-badge" :style="{ background: candidate.color }">{{ String(index + 1).padStart(2, "0") }}</div>

            <div class="cand-body">
              <div class="cand-top">
                <div>
                  <div class="name-row">
                    <h3>{{ candidate.name }}</h3>
                    <span v-if="candidate.id === winner?.id && total > 0" class="winner-tag">★ UNGGUL</span>
                  </div>
                  <p class="tagline">{{ candidate.tagline }}</p>
                </div>
                <div class="vote-display">
                  <strong>{{ candidate.votes.toLocaleString("id-ID") }}</strong>
                  <span>{{ percent(candidate.votes) }}%</span>
                </div>
              </div>

              <div class="progress-wrap">
                <div class="progress-track">
                  <div class="progress-fill" :style="{ width: percent(candidate.votes) + '%', background: candidate.color }"></div>
                </div>
              </div>

              <div class="cand-controls">
                <button class="btn minus-btn" @click="change(candidate, -1)">−</button>
                <button class="btn plus-btn" :style="{ '--clr': candidate.color }" @click="change(candidate, 1)">+</button>
              </div>
            </div>
          </article>
        </div>

        <aside class="sidebar">
          <div class="panel neu">
            <div class="section-head tight">
              <div>
                <p class="eyebrow">RINGKASAN</p>
                <h2>Grafik Suara</h2>
              </div>
            </div>
            <div v-for="cand in ranked" :key="cand.id" class="chart-row">
              <div class="chart-meta">
                <span>{{ cand.name }}</span>
                <strong>{{ cand.votes }} <small>{{ percent(cand.votes) }}%</small></strong>
              </div>
              <div class="chart-track">
                <div class="chart-fill" :style="{ width: percent(cand.votes) + '%', background: cand.color }"></div>
              </div>
            </div>
            <div class="total-line">
              <span>Total suara masuk</span>
              <strong>{{ counted }}</strong>
            </div>
          </div>
        </aside>
      </section>

      <section v-else class="panel neu log-panel">
        <div class="section-head">
          <div>
            <p class="eyebrow">AUDIT TRAIL</p>
            <h2>Riwayat Input Suara</h2>
          </div>
          <button class="btn undo-btn" :disabled="!logs.length" @click="undo">↶ Undo</button>
        </div>
        <div v-if="logs.length" class="log-list">
          <div v-for="entry in logs" :key="entry.id" class="log-item">
            <span class="log-time">{{ entry.time }}</span>
            <span class="log-label">{{ entry.label }}</span>
            <b :class="entry.amount > 0 ? 'plus' : 'minus'">
              {{ entry.amount > 0 ? "+" : "" }}{{ entry.amount }}
            </b>
          </div>
        </div>
        <div v-else class="empty-state">Belum ada input suara.</div>
      </section>
    </main>

    <div v-if="isFullscreen" class="fullscreen-controls">
      <button class="btn" @click="fullscreen">Keluar fullscreen</button>
      <button class="btn manage-btn" @click="openManage">Kelola paslon</button>
      <button class="btn danger" @click="showReset = true">Reset data</button>
    </div>

    <Teleport to="body">
      <div v-if="showManage" class="modal-backdrop" @click.self="showManage = false">
        <div class="modal neu manage-modal">
          <div class="section-head">
            <div><p class="eyebrow">PENGATURAN</p><h2>Kelola Paslon</h2></div>
            <button class="btn ghost" @click="showManage = false">Tutup</button>
          </div>
          <div class="candidate-editor">
            <div v-for="(candidate, index) in draftCandidates" :key="candidate.id" class="editor-row">
              <input v-model="candidate.name" aria-label="Nama paslon" placeholder="Nama paslon" />
              <input v-model="candidate.tagline" aria-label="Deskripsi paslon" placeholder="Deskripsi singkat" />
              <input v-model="candidate.color" type="color" aria-label="Warna paslon" />
              <button class="btn remove-btn" @click="removeCandidate(index)" :disabled="draftCandidates.length <= 1">Hapus</button>
            </div>
          </div>
          <div class="manage-actions">
            <button class="btn" @click="addCandidate">+ Tambah paslon</button>
            <button class="btn save-btn" @click="saveCandidates">Simpan perubahan</button>
          </div>
        </div>
      </div>
      <div v-if="showReset" class="modal-backdrop" @click.self="showReset = false">
        <div class="modal neu">
          <div class="modal-icon">!</div>
          <h2>Yakin mau hapus semua?</h2>
          <p>Semua perolehan suara dan riwayat log akan dihapus permanen dari perangkat ini. Tindakan ini tidak bisa di-undo.</p>
          <div class="modal-actions">
            <button class="btn ghost" @click="showReset = false">Batal</button>
            <button class="btn danger" @click="resetAll">Ya, hapus semua</button>
          </div>
        </div>
      </div>
    </Teleport>
  </div>
</template>
