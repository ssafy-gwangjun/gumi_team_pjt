<template>
  <section class="map-page-shell">
    <div class="map-header">
      <h2>🗺️로컬 여행 지도</h2>
      <p>
        금오산, 문화시설, 맛집, 숙박까지 구미의 주요 관광 정보를 지도에서 빠르게 확인하세요.
      </p>
    </div>

    <div class="map-grid">
      <div class="map-panel">
        <div class="filter-bar">
          <div class="filter-group filter-group-primary">
            <button
              v-for="type in primaryContentTypes"
              :key="type.id"
              @click="setContentType(type.id)"
              :class="['filter-pill', { active: type.id === contentTypeId }]"
            >
              <span class="pill-icon">{{ filterIcons[type.id] }}</span>
              {{ type.label }}
            </button>
          </div>

          <div v-if="secondaryContentTypes.length" class="filter-group filter-group-scroll">
            <button
              v-for="type in secondaryContentTypes"
              :key="type.id"
              @click="setContentType(type.id)"
              :class="['filter-pill', { active: type.id === contentTypeId }]"
            >
              <span class="pill-icon">{{ filterIcons[type.id] }}</span>
              {{ type.label }}
            </button>
          </div>
        </div>

        <div class="leaflet-wrap">
          <div id="map" class="map"></div>

          <div class="map-controls">
            <button type="button" @click="zoomIn" title="확대">+</button>
            <button type="button" @click="zoomOut" title="축소">−</button>
            <button type="button" @click="resetMap" title="구미 중심">⟲</button>
          </div>
        </div>
      </div>

      <aside class="side-panel">
        <div class="card-header">
          <div>
            <p class="panel-title">추천 장소</p>
            <p class="panel-subtitle">{{ filteredPlaces.length }}곳 표시 중 · {{ activeFilterLabel }}</p>
          </div>
          <div class="search-box">
            <input
              v-model="searchQuery"
              type="search"
              placeholder="장소명 / 주소 검색"
            />
          </div>
        </div>

        <div class="poi-list">
          <article
            v-for="place in filteredPlaces"
            :key="place.contentid"
            class="poi-card"
            :class="{ selected: place.contentid === activePlaceId }"
            @click="goToPlace(place)"
          >
            <div class="poi-thumb">
              <img
                :src="place.firstimage2 || place.firstimage || placeholderImage"
                :alt="place.title"
                @error="handleImageError"
              />
            </div>

            <div class="poi-info">
              <span class="poi-type">{{ getTypeLabel(place.contenttypeid) }}</span>
              <h3>{{ place.title }}</h3>
              <p>{{ place.addr1 }}</p>

              <div class="action-group">
                <button
                  class="action-btn like"
                  :class="{ active: isLiked(place) }"
                  @click.stop="toggleLike(place)"
                >
                  <span v-if="isLiked(place)">♥</span>
                  <span v-else>♡</span>
                  좋아요
                </button>
                <button
                  class="action-btn bookmark"
                  :class="{ active: isBookmarked(place) }"
                  @click.stop="toggleBookmark(place)"
                >
                  <span v-if="isBookmarked(place)">★</span>
                  <span v-else>☆</span>
                  북마크
                </button>
              </div>
            </div>
          </article>

          <div v-if="filteredPlaces.length === 0" class="empty-state">
            검색 결과가 없습니다.
          </div>
        </div>
      </aside>
    </div>
  </section>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import 'leaflet/dist/leaflet.css'
import L from 'leaflet'

const props = defineProps({
  bookmarkedIds: {
    type: Array,
    default: () => []
  },
  likedIds: {
    type: Array,
    default: () => []
  }
})

const emit = defineEmits(['toggle-bookmark', 'toggle-like'])

const GUMI_CENTER = { lat: 36.1119, lng: 128.3875 }
const GUMI_ZOOM = 12
const placeholderImage = 'https://placehold.co/420x280/e2e8f0/64748b?text=No+Image'

const map = ref(null)
const markerLayer = ref(null)
const allPlaces = ref([])
const contentTypeId = ref('15')
const searchQuery = ref('')
const activePlaceId = ref(null)

const contentTypes = [
  { id: 'all', label: '전체' },
  { id: '12', label: '관광지' },
  { id: '14', label: '문화시설' },
  { id: '15', label: '축제/공연' },
  { id: '25', label: '여행코스' },
  { id: '28', label: '레포츠' },
  { id: '32', label: '숙박' },
  { id: '38', label: '쇼핑' },
  { id: '39', label: '음식점' }
]

const primaryContentTypes = computed(() =>
  contentTypes.filter(type => ['all', '12', '15', '39'].includes(type.id))
)

const secondaryContentTypes = computed(() =>
  contentTypes.filter(type => !['all', '12', '15', '39'].includes(type.id))
)

const filterIcons = {
  all: '🌐',
  '12': '📍',
  '14': '🏛️',
  '15': '🎉',
  '25': '🧭',
  '28': '🚴',
  '32': '🛏️',
  '38': '🛍️',
  '39': '🍽️'
}

const categoryMarkerColors = {
  all: '#10b981',
  '12': '#3b82f6',
  '14': '#8b5cf6',
  '15': '#ec4899',
  '25': '#f59e0b',
  '28': '#14b8a6',
  '32': '#6366f1',
  '38': '#f97316',
  '39': '#ef4444'
}

const activeFilterLabel = computed(() => {
  const found = contentTypes.find((type) => type.id === contentTypeId.value)
  return found ? found.label : '전체'
})

const filteredPlaces = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()
  return allPlaces.value
    .filter((place) => {
      return contentTypeId.value === 'all' || place.contenttypeid === contentTypeId.value
    })
    .filter((place) => {
      if (!query) return true
      return (
        place.title?.toLowerCase().includes(query) ||
        place.addr1?.toLowerCase().includes(query) ||
        place.addr2?.toLowerCase().includes(query)
      )
    })
})

const normalizedLikedIds = computed(() => props.likedIds.map((id) => String(id)))
const normalizedBookmarkedIds = computed(() => props.bookmarkedIds.map((id) => String(id)))

const isLiked = (place) => normalizedLikedIds.value.includes(String(place.contentid))
const isBookmarked = (place) => normalizedBookmarkedIds.value.includes(String(place.contentid))

const getTypeLabel = (typeId) => {
  const found = contentTypes.find((type) => type.id === typeId)
  return found ? found.label : '기타'
}

const createMarkerIcon = (typeId) => {
  const color = categoryMarkerColors[typeId] || categoryMarkerColors.all

  const svg = `
    <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 36" width="28" height="42">
      <path
        d="M12 0C6.477 0 2 4.477 2 10c0 7.5 10 24 10 24s10-16.5 10-24c0-5.523-4.477-10-10-10z"
        fill="${color}"
        stroke="#ffffff"
        stroke-width="2"
      />
      <circle cx="12" cy="10" r="4.5" fill="#ffffff" />
    </svg>
  `

  return L.divIcon({
    className: 'custom-map-marker',
    html: svg,
    iconSize: [28, 42],
    iconAnchor: [14, 42],
    popupAnchor: [0, -36]
  })
}

const loadFile = async (fileName) => {
  const res = await fetch(`/data/${fileName}`)
  if (!res.ok) {
    throw new Error(`파일 로드 실패: ${fileName} (${res.status})`)
  }
  const data = await res.json()
  allPlaces.value.push(...(data.items || []))
}

const loadData = async () => {
  allPlaces.value = []
  const initialFiles = ['구미_경북권_축제공연행사.json']
  const laterFiles = [
    '구미_경북권_관광지.json',
    '구미_경북권_문화시설.json',
    '구미_경북권_레포츠.json',
    '구미_경북권_숙박.json',
    '구미_경북권_쇼핑.json',
    '구미_경북권_음식점.json',
    '구미_경북권_여행코스.json'
  ]

  try {
    for (const fileName of initialFiles) {
      await loadFile(fileName)
    }
  } catch (error) {
    console.error('초기 데이터 로드 실패:', error)
  }

  const loadRemaining = async () => {
    for (const fileName of laterFiles) {
      try {
        await loadFile(fileName)
      } catch (error) {
        console.error('추가 데이터 로드 실패:', error)
      }
    }
  }

  if ('requestIdleCallback' in window) {
    requestIdleCallback(loadRemaining)
  } else {
    setTimeout(loadRemaining, 100)
  }
}

const setContentType = (typeId) => {
  contentTypeId.value = typeId
}

const toggleLike = (place) => {
  emit('toggle-like', place)
}

const toggleBookmark = (place) => {
  emit('toggle-bookmark', place)
}

const updateMarkers = () => {
  if (!map.value) return
  if (!markerLayer.value) markerLayer.value = L.layerGroup().addTo(map.value)

  markerLayer.value.clearLayers()

  filteredPlaces.value.forEach((place) => {
    const lat = parseFloat(place.mapy)
    const lng = parseFloat(place.mapx)
    if (!Number.isFinite(lat) || !Number.isFinite(lng)) return

    const popupHtml = `
      <div style="max-width:260px; line-height:1.4; font-size:0.95rem;">
        <strong style="display:block; margin-bottom:8px;">${place.title || ''}</strong>
        <div style="margin-bottom:8px; color:#334155;">${place.addr1 || ''}</div>
      </div>
    `
    const marker = L.marker([lat, lng], { icon: createMarkerIcon(place.contenttypeid) })
      .bindPopup(popupHtml)
      .addTo(markerLayer.value)

    marker.on('click', () => {
      activePlaceId.value = place.contentid
    })
  })
}

const resetMap = () => {
  if (!map.value) return
  map.value.setView([GUMI_CENTER.lat, GUMI_CENTER.lng], GUMI_ZOOM)
  activePlaceId.value = null
}

const zoomIn = () => map.value?.zoomIn()
const zoomOut = () => map.value?.zoomOut()

const goToPlace = (place) => {
  const lat = parseFloat(place.mapy)
  const lng = parseFloat(place.mapx)
  if (!map.value || !Number.isFinite(lat) || !Number.isFinite(lng)) return
  map.value.flyTo([lat, lng], 15, { duration: 0.8 })
  activePlaceId.value = place.contentid
}

const handleImageError = (event) => {
  event.target.src = placeholderImage
}

onMounted(async () => {
  map.value = L.map('map', {
    center: [GUMI_CENTER.lat, GUMI_CENTER.lng],
    zoom: GUMI_ZOOM
  })

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution:
      '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>'
  }).addTo(map.value)

  await loadData()
})

watch(filteredPlaces, updateMarkers)
</script>

<style scoped>
.map-page-shell {
  display: grid;
  gap: 0.5rem;
  width: 100%;
  padding: 1rem;
  background: #f8fafc;
  border-radius: 24px;
  box-shadow: 0 24px 60px rgba(15, 23, 42, 0.08);
}

.map-header {
  margin-bottom: 0.5rem;
  padding: 1.25rem 1.5rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 16px;
}

.map-header h2 {
  margin: 0;
  font-size: 1.4rem;
  color: #111827;
}

.map-header p {
  margin: 0.5rem 0 0;
  color: #555;
}

.page-hero {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 1rem;
  padding: 1.2rem 1.3rem;
  background: linear-gradient(135deg, #0f766e, #22c55e);
  border-radius: 24px;
  color: white;
}

.hero-copy {
  max-width: 720px;
}

.hero-badge {
  display: inline-flex;
  align-items: center;
  gap: 0.45rem;
  margin-bottom: 0.7rem;
  padding: 0.5rem 0.85rem;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.18);
  font-size: 0.78rem;
  text-transform: uppercase;
  letter-spacing: 0.1em;
}

.hero-copy h2 {
  margin: 0;
  font-size: clamp(1.9rem, 2.8vw, 2.6rem);
  line-height: 1.05;
}

.hero-copy p {
  margin: 0.9rem 0 0;
  font-size: 1rem;
  line-height: 1.75;
  opacity: 0.95;
  max-width: 680px;
}

.map-grid {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 1rem;
}

.map-panel {
  display: grid;
  gap: 1rem;
}

.filter-bar {
  background: white;
  border-radius: 20px;
  padding: 1rem 1.1rem;
  box-shadow: 0 16px 36px rgba(15, 23, 42, 0.05);
}

.filter-group {
  display: flex;
  gap: 0.65rem;
  flex-wrap: wrap;
}

.filter-group-primary {
  margin-bottom: 0.25rem;
}

.filter-group-scroll {
  display: flex;
  flex-wrap: nowrap;
  overflow-x: auto;
  padding-bottom: 0.25rem;
  scrollbar-width: thin;
  scrollbar-color: #cbd5e1 transparent;
}

.filter-group-scroll::-webkit-scrollbar {
  height: 6px;
}

.filter-group-scroll::-webkit-scrollbar-thumb {
  background: #cbd5e1;
  border-radius: 999px;
}

.filter-pill {
  flex: 0 0 auto;
  border: 1px solid transparent;
  border-radius: 999px;
  padding: 0.85rem 1rem;
  background: #f8fafc;
  color: #334155;
  cursor: pointer;
  transition: all 0.18s ease;
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.92rem;
}

.filter-pill.active {
  background: #0f766e;
  color: white;
  box-shadow: 0 12px 28px rgba(15, 23, 42, 0.16);
}

.pill-icon {
  width: 1.4rem;
  height: 1.4rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.leaflet-wrap {
  position: relative;
  min-height: 560px;
  border-radius: 24px;
  overflow: hidden;
  border: 1px solid #e2e8f0;
  background: white;
  box-shadow: 0 24px 50px rgba(15, 23, 42, 0.08);
}

.map {
  width: 100%;
  height: 100%;
}

.map-controls {
  position: absolute;
  right: 18px;
  bottom: 18px;
  display: grid;
  gap: 0.65rem;
}

.map-controls button {
  width: 42px;
  height: 42px;
  border-radius: 14px;
  border: none;
  background: white;
  color: #0f172a;
  box-shadow: 0 14px 30px rgba(15, 23, 42, 0.12);
  cursor: pointer;
  font-size: 1.1rem;
}

.side-panel {
  display: grid;
  gap: 1rem;
}

.card-header {
  display: grid;
  gap: 0.85rem;
  padding: 1rem 1.1rem;
  border-radius: 22px;
  background: white;
  box-shadow: 0 16px 36px rgba(15, 23, 42, 0.05);
}

.panel-title {
  margin: 0;
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
}

.panel-subtitle {
  margin: 0.3rem 0 0;
  color: #475569;
  font-size: 0.92rem;
}

.search-box input {
  width: 100%;
  border: 1px solid #cbd5e1;
  border-radius: 16px;
  padding: 0.95rem 1rem;
  font-size: 0.95rem;
  outline: none;
  background: #f8fafc;
  color: #0f172a;
}

.poi-list {
  display: grid;
  gap: 0.9rem;
  max-height: 640px;
  overflow-y: auto;
  padding-right: 4px;
}

.poi-card {
  display: grid;
  grid-template-columns: 120px 1fr;
  gap: 0.85rem;
  padding: 1rem;
  border-radius: 22px;
  background: white;
  border: 1px solid transparent;
  cursor: pointer;
  transition: all 0.18s ease;
  box-shadow: 0 14px 30px rgba(15, 23, 42, 0.05);
}

.poi-card:hover,
.poi-card.selected {
  transform: translateY(-1px);
  border-color: #a7f3d0;
  box-shadow: 0 18px 38px rgba(15, 23, 42, 0.12);
}

.poi-thumb {
  width: 100%;
  height: 110px;
  border-radius: 18px;
  overflow: hidden;
  background: #f1f5f9;
}

.poi-thumb img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.poi-info {
  display: grid;
  gap: 0.7rem;
}

.poi-type {
  display: inline-flex;
  align-items: center;
  padding: 0.35rem 0.75rem;
  border-radius: 999px;
  background: #ecfdf5;
  color: #166534;
  font-size: 0.78rem;
  font-weight: 700;
  width: fit-content;
}

.poi-info h3 {
  margin: 0;
  font-size: 1rem;
  line-height: 1.35;
  color: #0f172a;
}

.poi-info p {
  margin: 0;
  font-size: 0.92rem;
  color: #475569;
  line-height: 1.55;
}

.action-group {
  display: flex;
  gap: 0.55rem;
  flex-wrap: wrap;
}

.action-btn {
  border: none;
  border-radius: 999px;
  padding: 0.7rem 0.9rem;
  font-size: 0.85rem;
  cursor: pointer;
  transition: all 0.18s ease;
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  color: #334155;
  background: #f8fafc;
}

.action-btn.like.active {
  background: #fee2e2;
  color: #991b1b;
}

.action-btn.bookmark.active {
  background: #ecfdf5;
  color: #047857;
}

.empty-state {
  padding: 2rem;
  border-radius: 24px;
  background: white;
  text-align: center;
  color: #64748b;
  border: 1px dashed #cbd5e1;
}

.custom-map-marker .marker-pin {
  width: 18px;
  height: 18px;
  display: inline-block;
  border-radius: 50%;
  border: 2px solid white;
  box-shadow: 0 0 0 4px rgba(0, 0, 0, 0.06), 0 4px 10px rgba(15, 23, 42, 0.16);
}

@media (max-width: 1200px) {
  .map-grid {
    grid-template-columns: 1fr;
  }
}

@media (max-width: 860px) {
  .page-hero {
    flex-direction: column;
  }

  .poi-card {
    grid-template-columns: 1fr;
  }

  .map-controls {
    right: 12px;
    bottom: 12px;
  }
}
</style>