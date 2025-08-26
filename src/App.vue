<!-- src/App.vue -->
<template>
  <div class="kid-page">
    <!-- HERO / BUSCADOR -->
    <section class="hero">
      <div class="hero-inner">
        <div class="brand">
          <span class="emoji">🧠</span>
          <h1>MiniDiccionario</h1>
          <span class="spark">✨</span>
        </div>
        <p class="subtitle">Busca conceptos de Física con ejemplos fáciles</p>

        <div class="search-wrap">
          <div class="search">
            <span class="icon">🔍</span>
            <input
              v-model="query"
              class="search-input"
              :placeholder="placeholder"
              @keydown.enter.prevent
            />
            <button v-if="query" class="clear" @click="query = ''" aria-label="Limpiar">✕</button>
          </div>
          <div class="suggestions" v-if="suggested.length">
            <button
              v-for="s in suggested"
              :key="s"
              class="chip chip-suggestion"
              @click="query = s"
            >{{ s }}</button>
          </div>
        </div>

        <!-- FILTROS RÁPIDOS -->
        <div class="filters-quick">
          <div class="cats">
            <button
              v-for="c in categories"
              :key="c"
              :class="['chip', 'chip-cat', { active: category === c }]"
              @click="category = c"
            >
              <span class="emoji">{{ emojiForCategory(c) }}</span>{{ c }}
            </button>
          </div>
          <div class="toggles">
            <button class="chip chip-ghost" @click="sortAsc = !sortAsc">
              Orden: {{ sortAsc ? 'A→Z' : 'Z→A' }}
            </button>
            <button class="chip chip-ghost" @click="clearFilters">
              Limpiar filtros
            </button>
          </div>
        </div>

        <div class="tiny-info">
          <span class="pill-count">{{ filtered.length }}</span>
          <span> resultados de </span>
          <span class="pill-count">{{ data.length }}</span>
        </div>
      </div>
      <div class="bubbles" aria-hidden="true">
        <span class="b1"></span><span class="b2"></span><span class="b3"></span><span class="b4"></span>
      </div>
    </section>

    <!-- TAGS -->
    <section class="tags-area">
      <div class="tags-head">
        <h2>Filtrar por etiquetas</h2>
        <button v-if="selectedTags.length" class="link" @click="selectedTags = []">
          Limpiar ({{ selectedTags.length }})
        </button>
      </div>
      <div v-if="allTags.length" class="tag-cloud">
        <button
          v-for="t in allTags" :key="t"
          :class="['chip', 'chip-tag', { active: selectedTags.includes(t) }]"
          @click="toggleTag(t)"
        >
          <span class="emoji">{{ emojiForTag(t) }}</span>{{ t }}
        </button>
      </div>
      <p v-else class="muted">Todavía no hay etiquetas.</p>
    </section>

    <!-- RESULTADOS -->
    <main class="results">
      <div v-if="loading" class="empty">Cargando conceptos…</div>
      <div v-else-if="error" class="empty">Error al cargar: {{ error }}</div>
      <div v-else-if="!filtered.length" class="empty">
        <div class="big-emoji">🧐</div>
        <p>Sin resultados. Prueba con <strong>Fuerza</strong>, <strong>Gravedad</strong> o <strong>Movimiento</strong>.</p>
      </div>

      <article v-for="it in filtered" :key="it.term" class="card">
        <div class="card-art">
          <div class="sticker">{{ emojiForItem(it) }}</div>
          <img v-if="imageUrlForTerm(it.term)" class="thumb" :src="imageUrlForTerm(it.term)" :alt="`Imagen de ${it.term}`" />
        </div>

        <div class="card-body">
          <h3 class="title">
            <template v-for="(part, i) in highlightParts(it.term, query)" :key="i">
              <mark v-if="part.hit">{{ part.text }}</mark><span v-else>{{ part.text }}</span>
            </template>
          </h3>

          <div class="badges">
            <span class="badge badge-cat">
              {{ it.category || 'Sin categoría' }}
            </span>
            <span v-for="t in it.tags" :key="t" class="badge badge-tag" @click="toggleTag(t)">
              {{ t }}
            </span>
          </div>

          <p v-if="it.definicion_ninos" class="p kids">
            <strong>Para peques:</strong>
            <template v-for="(part, i) in highlightParts(it.definicion_ninos, query)" :key="i">
              <mark v-if="part.hit">{{ part.text }}</mark><span v-else>{{ part.text }}</span>
            </template>
          </p>

          <p v-if="it.ejemplo_ninos" class="p example">
            <strong>Ejemplo:</strong>
            <template v-for="(part, i) in highlightParts(it.ejemplo_ninos, query)" :key="i">
              <mark v-if="part.hit">{{ part.text }}</mark><span v-else>{{ part.text }}</span>
            </template>
          </p>

          <details v-if="it.definicion_formal" class="details">
            <summary>Ver definición formal</summary>
            <p class="p formal">
              <template v-for="(part, i) in highlightParts(it.definicion_formal, query)" :key="i">
                <mark v-if="part.hit">{{ part.text }}</mark><span v-else>{{ part.text }}</span>
              </template>
            </p>
          </details>
        </div>
      </article>
    </main>

    <footer class="foot">
      Hecho con ❤️ — datos desde <code>/conceptos.json</code>
    </footer>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue';

// Estado base
const data = ref([]);
const loading = ref(true);
const error = ref('');

// Controles UI
const query = ref('');
const category = ref('Todas');
const selectedTags = ref([]);
const sortAsc = ref(true);

// Carga JSON
onMounted(async () => {
  try {
    loading.value = true;
    const res = await fetch('/conceptos.json', { cache: 'no-store' });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    const raw = await res.json();
    data.value = raw.map(r => ({ ...r, tags: sanitizeTags(r.tags) }));
  } catch (e) {
    error.value = e.message || String(e);
  } finally {
    loading.value = false;
  }
});

// Normaliza tags
function sanitizeTags(tags) {
  if (!tags) return [];
  if (Array.isArray(tags)) {
    const out = [];
    for (const t of tags) {
      if (typeof t === 'string' && t.includes(',')) {
        for (const p of t.split(',')) {
          const s = p.trim();
          if (s && !out.includes(s)) out.push(s);
        }
      } else if (typeof t === 'string') {
        const s = t.trim();
        if (s && !out.includes(s)) out.push(s);
      }
    }
    return out;
  }
  if (typeof tags === 'string') {
    return tags.split(',').map(s => s.trim()).filter(Boolean);
  }
  return [];
}

// Categorías y tags
const categories = computed(() => {
  const s = new Set(['Todas']);
  for (const d of data.value) if (d.category) s.add(d.category);
  return Array.from(s);
});
const allTags = computed(() => {
  const s = new Set();
  for (const d of data.value) for (const t of d.tags || []) s.add(t);
  return Array.from(s).sort((a, b) => a.localeCompare(b, 'es', { sensitivity: 'base' }));
});

// Filtro y orden
const filtered = computed(() => {
  const q = query.value.trim().toLowerCase();
  const out = data.value.filter((item) => {
    if (category.value !== 'Todas' && (item.category || '') !== category.value) return false;
    if (selectedTags.value.length) {
      const ts = new Set(item.tags || []);
      const any = selectedTags.value.some(t => ts.has(t));
      if (!any) return false;
    }
    if (!q) return true;
    const fields = [
      item.term, item.category, ...(item.tags || []),
      item.definicion_ninos, item.ejemplo_ninos, item.definicion_formal
    ].filter(Boolean).map(String);
    return fields.some(f => f.toLowerCase().includes(q));
  });

  out.sort((a, b) =>
    sortAsc.value
      ? a.term.localeCompare(b.term, 'es', { sensitivity: 'base' })
      : b.term.localeCompare(a.term, 'es', { sensitivity: 'base' })
  );
  return out;
});

// UI helpers
const placeholder = computed(() => {
  const opts = [
    'Busca “Fuerza” 🔥', 'Prueba “Gravedad” 🍎', 'Escribe “Movimiento” 🛹',
    'Intenta “Palanca” ⚙️', 'Explora “Energía” 🔋'
  ];
  return opts[Math.floor(Date.now() / 5000) % opts.length];
});
const suggested = computed(() => {
  const hints = ['Fuerza', 'Gravedad', 'Movimiento', 'Palanca', 'Energía', 'MRU'];
  return hints.filter(h => data.value.some(d => d.term.toLowerCase().includes(h.toLowerCase()))).slice(0, 5);
});
function clearFilters() {
  query.value = '';
  category.value = 'Todas';
  selectedTags.value = [];
}
function toggleTag(t) {
  const i = selectedTags.value.indexOf(t);
  if (i >= 0) selectedTags.value.splice(i, 1);
  else selectedTags.value.push(t);
}
function emojiForCategory(c) {
  if (c === 'Todas') return '🧩';
  const map = {
    'Física': '🧪',
    'Mecánica clásica': '⚙️',
  };
  return map[c] || '📚';
}
function emojiForTag(t) {
  const k = t?.toLowerCase?.() || '';
  if (k.includes('cinemática')) return '🛹';
  if (k.includes('dinámica')) return '💥';
  if (k.includes('energía')) return '🔋';
  if (k.includes('rotación')) return '🌀';
  if (k.includes('equilibrio')) return '⚖️';
  if (k.includes('fluidos')) return '💧';
  if (k.includes('oscil')) return '🎵';
  if (k.includes('máquinas')) return '⚙️';
  return '🏷️';
}
function emojiForItem(it) {
  const ts = (it.tags || []).join(' ').toLowerCase();
  if (ts.includes('cinemática')) return '🛼';
  if (ts.includes('dinámica')) return '🚀';
  if (ts.includes('energía')) return '🔌';
  if (ts.includes('rotación')) return '🛰️';
  if (ts.includes('fluidos')) return '🌊';
  return '📘';
}
function escapeRegExp(s) {
  return s.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
function highlightParts(text, q) {
  if (!q) return [{ text, hit: false }];
  const rx = new RegExp(`(${escapeRegExp(q)})`, 'ig');
  return String(text).split(rx).map(p => ({ text: p, hit: p.toLowerCase() === q.toLowerCase() }));
}

// IMÁGENES (opcional): busca coincidencia por nombre de archivo
const images = import.meta.glob('./assets/imagenes/*.{png,jpg,jpeg,webp,svg}', { eager: true, as: 'url' });
function baseName(p) { return String(p).split('/').pop().split('.').slice(0, -1).join('.').toLowerCase(); }
function strip(s) { return s.normalize('NFD').replace(/[\u0300-\u036f]/g, '').toLowerCase().replace(/[^a-z0-9]+/g,''); }
function imageUrlForTerm(term) {
  const st = strip(term || '');
  let best = null;
  for (const [path, url] of Object.entries(images)) {
    const key = strip(baseName(path));
    if (key.includes(st) || st.includes(key)) { best = url; break; }
  }
  return best;
}
</script>

<style scoped>

.kid-page { background: var(--bg); color: var(--ink); min-height: 100vh; display: flex; flex-direction: column; }

/* Contenedores principales */
.hero-inner, .tags-area, .results, .foot {
  width: min(1400px, 100%);
  margin: 0 auto;
}

.tags-area  { margin-top: 8px; padding: 0 16px; }
.results {
  margin: 12px auto;
  padding: 0 16px 24px;
  display: grid;
  gap: 16px;
  /* columnas de ancho controlado */
  grid-template-columns: repeat(auto-fit, minmax(320px, 360px));
  justify-content: center; /* <-- centra la grilla */
}

.foot { padding: 16px; color: var(--muted); }

.search-wrap { display:flex; flex-direction: column; gap: 8px; }
/* ✅ buscador a todo el ancho */
.search {
  position: relative; display:flex; align-items:center; gap:8px;
  background: #fff; border: 2px solid #e5e7eb; border-radius: 16px; padding: 10px 12px;
  box-shadow: 0 6px 0 #dbeafe;
  width: 100%;
}
.icon { font-size: 18px; }
.search-input {
  flex: 1; border: none; outline: none; font-size: 16px; background: transparent;
}
.clear { border: none; background: #f3f4f6; border-radius: 10px; padding: 4px 8px; cursor: pointer; }

.suggestions { display:flex; flex-wrap:wrap; gap:8px; }
.chip {
  border: 2px solid #e5e7eb; background:#fff; padding: 6px 12px; border-radius: 999px; cursor: pointer;
  font-weight: 600; transition: transform .05s ease, box-shadow .15s ease; user-select: none;
}
.chip:active { transform: translateY(1px); }
.chip-suggestion { border-color:#c7d2fe; background: #eef2ff; }
.chip-ghost { background: #fff; }
.chip-cat { background: #fff; }
.chip-tag { background: #fff; }

.filters-quick {  display: flex;  gap: 12px;  align-items: center;  justify-content: center;  /* ← centrado */  flex-wrap: wrap;          /* ← que pueda saltar en pantallas chicas */  margin-top: 10px;}
.cats {  display: flex;  gap: 8px;  overflow: auto;  padding-bottom: 4px;  justify-content: center;   /* <-- centrado */}
.cats .chip.active { border-color:#60a5fa; box-shadow: 0 3px 0 #dbeafe; }
.emoji { margin-right: 6px; }

.toggles { display:flex; gap:8px; }

.tiny-info { margin-top: 8px; color: var(--muted); font-size: 14px; }
.pill-count { background:#fff; border: 2px solid #e5e7eb; border-radius: 999px; padding: 2px 8px; }


.bubbles span { position: absolute; width: 14px; height: 14px; background: #bfdbfe; border-radius: 999px; opacity: .5; animation: float 8s linear infinite; }
.b1 { left: 6%; top: 60%; animation-delay: 0s; }
.b2 { left: 20%; top: 70%; animation-delay: 1s; }
.b3 { right: 16%; top: 55%; animation-delay: 2s; background: #fbcfe8; }
.b4 { right: 8%; top: 75%; animation-delay: 3s; background: #fde68a; }
@keyframes float { 0% { transform: translateY(0) } 50% { transform: translateY(-10px) } 100% { transform: translateY(0) } }

/* TAGS */
.tags-head { display:flex; align-items: center; justify-content: space-between; }
.link { background:none; border:none; color:#2563eb; text-decoration: underline; cursor: pointer; }
.tag-cloud {  display: flex;  flex-wrap: wrap;  gap: 8px;  margin-top: 8px;  justify-content: center;   /* <-- centrado */}

/* TARJETAS */
.card {
  display:grid; grid-template-columns: 96px 1fr; gap: 12px;
  background: #ffffff; border: 2px solid #e5e7eb; border-radius: 18px; padding: 12px;
  box-shadow: 0 6px 0 #ffe4e6;
}
.card-art { position: relative; display:flex; align-items:center; justify-content:center; }
.sticker {
  font-size: 36px; background: #fff7ed; border: 2px solid #fed7aa; border-radius: 14px; padding: 8px 12px;
  box-shadow: 0 4px 0 #fdba74;
}
.thumb {
  position: absolute; right: -8px; bottom: -10px; width: 56px; height: 56px; object-fit: cover;
  border-radius: 12px; border: 2px solid #e5e7eb; background:#fff;
}
.card-body { display:flex; flex-direction: column; gap: 6px; }
.title { margin: 0; font-weight: 800; font-size: 18px; }
.badges { display:flex; flex-wrap:wrap; gap: 6px; }
.badge {
  font-size: 11px; padding: 4px 8px; border-radius: 999px; border: 2px solid #e5e7eb; background: #fff;
}
.badge-cat { background: var(--badge); border-color: #c7d2fe; color: var(--badge-ink); }
.badge-tag { cursor: pointer; }
.p { margin: 0; line-height: 1.45; }
.p.kids { background: #f0f9ff; border: 2px solid #bae6fd; border-radius: 10px; padding: 6px 8px; }
.p.example { background: #fef9c3; border: 2px solid #fde68a; border-radius: 10px; padding: 6px 8px; }
.p.formal { margin-top: 6px; }
.details summary { cursor: pointer; margin-top: 4px; }

.empty {
  grid-column: 1 / -1; text-align: center; padding: 36px; border: 2px dashed #e5e7eb; border-radius: 16px; color: var(--muted); background: #fff;
}
.big-emoji { font-size: 40px; margin-bottom: 6px; }

mark { background: #fde68a; border-radius: 4px; padding: 0 2px; }
.muted { color: var(--muted); }

/* HERO — visible, centrado y por encima de las burbujas */
.hero{
  position: relative;
  isolation: isolate;             /* crea contexto para z-index propios */
  overflow: hidden;               /* recorta burbujas que se salgan */
  padding: 24px 0 12px;
  min-height: 170px;              /* asegura que el buscador siempre tenga espacio */
  background:
    radial-gradient(1200px 300px at 20% -10%, #dbeafe, transparent),
    radial-gradient(900px 250px at 80% -20%, #ffe4e6, transparent),
    radial-gradient(700px 200px at 50% 0%, #fef9c3, transparent);
}

.hero-inner{
  position: relative;
  z-index: 1;                     /* contenido por encima de .bubbles */
  width: min(1400px, 100%);
  margin: 0 auto;
  padding: 28px 16px 12px;        /* conserva tu padding */
}

.bubbles{
  position: absolute;
  inset: 0;
  z-index: 0;                     /* burbujas detrás del contenido */
  pointer-events: none;
}

/* márgenes pequeños para evitar colapsos visuales */
.brand h1{ margin: 0; line-height: 1.2; }
.subtitle{ margin: 8px 0 12px; color: var(--muted); }



</style>

<style>

/* Global (no scoped) */
:root {
  --bg: #f9fbff;
  --ink: #0f172a;
  --muted: #475569;
  --brand-1: #7c9cf7;
  --brand-2: #f9a8d4;
  --brand-3: #fde68a;
  --ok: #10b981;
  --badge: #eef2ff;
  --badge-ink: #4338ca;
}

html, body, #app {
  height: 100%;
  background: var(--bg);
  color: var(--ink);
  margin: 0; /* quita margen por defecto del navegador */
}

/* Anula el max-width/padding del template por defecto */
#app {
  max-width: none !important;
  width: 100% !important;
  padding: 0 !important;
  margin: 0 !important;
}
</style>
