<template>
  <section class="map-container">
    <header class="map-header">
      <div>
        <h2>🗺️ 구미 관광지도</h2>
        <p>검색어와 유형 필터로 구미 관광지를 찾아보세요.</p>
      </div>

      <div class="map-controls-top">
        <input
          v-model="searchQuery"
          type="search"
          placeholder="장소명으로 검색하기"
        />

        <select v-model="contentTypeId">
          <option
            v-for="type in contentTypes"
            :key="type.id"
            :value="type.id"
          >
            {{ type.label }}
          </option>
        </select>

        <button type="button" @click="resetMap">
          구미 중심 재정렬
        </button>
      </div>
    </header>

    <div class="map-grid">
      <div class="leaflet-wrap">
        <div id="map" class="map"></div>

        <div class="map-controls">
          <button type="button" @click="zoomIn" title="확대">+</button>
          <button type="button" @click="zoomOut" title="축소">−</button>
          <button type="button" @click="resetMap" title="구미 중심">
            🔄
          </button>
        </div>
      </div>

      <aside class="place-list">
        <div class="list-header">
          <strong>{{ filteredPlaces.length }}개 장소</strong>
          <span>{{ activeFilterLabel }}</span>
        </div>

        <div v-if="filteredPlaces.length === 0" class="empty-state">
          검색 결과가 없습니다.
        </div>

        <ul>
          <li
            v-for="place in filteredPlaces"
            :key="place.contentid"
            :class="{ active: place.contentid === activePlaceId }"
            @click="goToPlace(place)"
          >
            <div class="thumb">
              <img
                :src="place.firstimage2 || place.firstimage || placeholderImage"
                :alt="place.title"
              />
            </div>

            <div class="info">
              <h3>{{ place.title }}</h3>
              <p>{{ place.addr1 }} {{ place.addr2 }}</p>

              <div class="place-actions">
                <button
                  class="action like"
                  :class="{ on: isLiked(place) }"
                  @click.stop="toggleLike(place)"
                  :aria-pressed="isLiked(place)"
                  title="좋아요"
                >
                  <span v-if="isLiked(place)">♥</span>
                  <span v-else>♡</span>
                </button>

                <button
                  class="action bookmark"
                  :class="{ on: isBookmarked(place) }"
                  @click.stop="toggleBookmark(place)"
                  :aria-pressed="isBookmarked(place)"
                  title="북마크"
                >
                  <span v-if="isBookmarked(place)">★</span>
                  <span v-else>☆</span>
                </button>
              </div>
            </div>
          </li>
        </ul>
      </aside>
    </div>
  </section>
</template>

<script setup>
import { ref, computed, onMounted, watch } from 'vue'
import 'leaflet/dist/leaflet.css'
import L from 'leaflet'
import iconUrl from 'leaflet/dist/images/marker-icon.png'
import iconRetinaUrl from 'leaflet/dist/images/marker-icon-2x.png'
import iconShadowUrl from 'leaflet/dist/images/marker-shadow.png'

const props = defineProps({
  bookmarkedIds: {
    type: Array,
    default: () => [],
  },
  likedIds: {
    type: Array,
    default: () => [],
  },
})

const emit = defineEmits(['toggle-bookmark', 'toggle-like'])

const GUMI_CENTER = { lat: 36.1119, lng: 128.3875 }
const GUMI_ZOOM = 12
const placeholderImage = 'https://via.placeholder.com/240x160?text=No+Image'

const map = ref(null)
const markerLayer = ref(null)
const allPlaces = ref([])
const contentTypeId = ref('all')
const searchQuery = ref('')
const activePlaceId = ref(null)

const contentTypes = [
  { id: 'all', label: '전체' },
  { id: '12', label: '관광지' },
  { id: '14', label: '문화시설' },
  { id: '15', label: '축제공연행사' },
  { id: '25', label: '여행코스' },
  { id: '28', label: '레포츠' },
  { id: '32', label: '숙박' },
  { id: '38', label: '쇼핑' },
  { id: '39', label: '음식점' },
]

const activeFilterLabel = computed(() => {
  const found = contentTypes.find((type) => type.id === contentTypeId.value)
  return found ? found.label : '전체'
})

const filteredPlaces = computed(() => {
  const query = searchQuery.value.trim().toLowerCase()

  return allPlaces.value
    .filter((place) => {
      return (
        contentTypeId.value === 'all' ||
        place.contenttypeid === contentTypeId.value
      )
    })
    .filter((place) => {
      if (!query) return true
      return place.title?.toLowerCase().includes(query)
    })
})

const normalizedLikedIds = computed(() =>
  props.likedIds.map((id) => String(id))
)
const normalizedBookmarkedIds = computed(() =>
  props.bookmarkedIds.map((id) => String(id))
)

const isLiked = (place) =>
  normalizedLikedIds.value.includes(String(place.contentid))
const isBookmarked = (place) =>
  normalizedBookmarkedIds.value.includes(String(place.contentid))

const defaultIcon = L.icon({
  iconUrl,
  iconRetinaUrl,
  shadowUrl: iconShadowUrl,
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41],
})

L.Marker.prototype.options.icon = defaultIcon

const getDataUrl = (fileName) => `/data/${fileName}`

const loadData = async () => {
  const dataFiles = [
    '구미_경북권_관광지.json',
    '구미_경북권_문화시설.json',
    '구미_경북권_축제공연행사.json',
    '구미_경북권_레포츠.json',
    '구미_경북권_숙박.json',
    '구미_경북권_쇼핑.json',
    '구미_경북권_음식점.json',
    '구미_경북권_여행코스.json',
  ]

  const responses = await Promise.all(
    dataFiles.map(async (fileName) => {
      const url = getDataUrl(fileName)
      const res = await fetch(url)
      if (!res.ok) {
        throw new Error(`파일 로드 실패: ${url} (${res.status})`)
      }
      return res.json()
    })
  )

  allPlaces.value = responses.flatMap((data) => data.items || [])
}

const toggleLike = (place) => {
  emit('toggle-like', place)
}

const toggleBookmark = (place) => {
  emit('toggle-bookmark', place)
}

const updateMarkers = () => {
  if (!map.value) return
  if (!markerLayer.value) {
    markerLayer.value = L.layerGroup().addTo(map.value)
  }

  markerLayer.value.clearLayers()

  filteredPlaces.value.forEach((place) => {
    const lat = parseFloat(place.mapy)
    const lng = parseFloat(place.mapx)
    if (!Number.isFinite(lat) || !Number.isFinite(lng)) return

    const imageSrc = place.firstimage || place.firstimage2 || placeholderImage
    const popupHtml = `
      <div style="max-width:280px">
        <img src="${imageSrc}" alt="${place.title || ''}"
          style="width:100%;height:140px;object-fit:cover;border-radius:8px;margin-bottom:8px"/>
        <strong style="display:block;margin-bottom:4px">${place.title || ''}</strong>
        <div style="color:#444;font-size:0.95rem">${place.addr1 || ''}</div>
      </div>
    `

    const marker = L.marker([lat, lng], { icon: defaultIcon })
      .bindPopup(popupHtml)
      .addTo(markerLayer.value)

    marker.on('click', () => {
      activePlaceId.value = place.contentid
    })
  })
}

const goToPlace = (place) => {
  const lat = parseFloat(place.mapy)
  const lng = parseFloat(place.mapx)
  if (!map.value || !Number.isFinite(lat) || !Number.isFinite(lng)) return

  map.value.flyTo([lat, lng], 15, { duration: 0.8 })
  activePlaceId.value = place.contentid

  L.popup({ maxWidth: 320 })
    .setLatLng([lat, lng])
    .setContent(
      `<div style="max-width:320px">
         <img src="${place.firstimage || place.firstimage2 || placeholderImage}" style="width:100%;height:140px;object-fit:cover;border-radius:8px;margin-bottom:8px"/>
         <strong>${place.title}</strong><br>${place.addr1 || ''}
       </div>`
    )
    .openOn(map.value)
}

const resetMap = () => {
  if (!map.value) return
  map.value.setView([GUMI_CENTER.lat, GUMI_CENTER.lng], GUMI_ZOOM)
}

const zoomIn = () => {
  if (!map.value) return
  map.value.zoomIn()
}

const zoomOut = () => {
  if (!map.value) return
  map.value.zoomOut()
}

onMounted(async () => {
  map.value = L.map('map', {
    center: [GUMI_CENTER.lat, GUMI_CENTER.lng],
    zoom: GUMI_ZOOM,
  })

  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution:
      '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a>',
  }).addTo(map.value)

  try {
    await loadData()
    updateMarkers()
  } catch (error) {
    console.error('지도 데이터 로드 실패:', error)
  }
})

watch(filteredPlaces, updateMarkers)
</script>

<style scoped>
.map-container {
  width: calc(100% - 2rem);
  margin: 0 auto;
  display: grid;
  gap: 1rem;
  padding: 1rem;
  background: #ffffff;
  border-radius: 16px;
  box-shadow: 0 18px 40px rgba(0, 0, 0, 0.08);
}

.map-header {
  display: flex;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 1rem;
  align-items: center;
}

.map-header h2 {
  margin: 0 0 0.25rem;
  font-size: 1.4rem;
}

.map-header p {
  margin: 0;
  color: #666;
}

.map-controls-top {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.map-controls-top input,
.map-controls-top select,
.map-controls-top button {
  border: 1px solid #d7d7d7;
  border-radius: 10px;
  padding: 0.85rem 1rem;
  font-size: 0.95rem;
}

.map-controls-top input {
  min-width: 220px;
  flex: 1;
}

.map-controls-top button {
  background: #2f6ae4;
  color: white;
  cursor: pointer;
}

.map-grid {
  display: grid;
  grid-template-columns: 2fr 0.9fr;
  gap: 1rem;
}

.leaflet-wrap {
  position: relative;
  min-height: 620px;
  border-radius: 16px;
  overflow: hidden;
  box-shadow: inset 0 0 0 1px rgba(0, 0, 0, 0.06);
}

.map {
  width: 100%;
  height: 100%;
}

.map-controls {
  position: absolute;
  top: 16px;
  right: 16px;
  display: grid;
  gap: 0.5rem;
}

.map-controls button {
  width: 42px;
  height: 42px;
  border: none;
  background: white;
  border-radius: 12px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.08);
  cursor: pointer;
  font-size: 1.2rem;
}

.place-list {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  max-height: 620px;
  overflow-y: auto;
  padding: 1rem;
  border-radius: 16px;
  background: #fafafa;
  border: 1px solid #eceff5;
}

.list-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 1rem;
}

.place-list ul {
  list-style: none;
  margin: 0;
  padding: 0;
  display: grid;
  gap: 0.75rem;
}

.place-list li {
  display: grid;
  grid-template-columns: 120px 1fr;
  gap: 0.85rem;
  padding: 0.85rem;
  border-radius: 14px;
  border: 1px solid transparent;
  cursor: pointer;
  transition: border-color 0.2s, transform 0.2s;
  background: #fff;
}

.place-list li:hover,
.place-list li.active {
  transform: translateY(-1px);
  border-color: #b2d2ff;
  background: #f7fbff;
}

.thumb {
  min-width: 120px;
  min-height: 90px;
  overflow: hidden;
  border-radius: 14px;
  background: #eceff5;
}

.thumb img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.info h3 {
  margin: 0 0 0.35rem;
  font-size: 1rem;
  line-height: 1.3;
}

.info p {
  margin: 0;
  color: #555;
  font-size: 0.95rem;
  line-height: 1.4;
  margin-bottom: 8px;
}

.place-actions {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.action {
  border: none;
  background: #eef2ff;
  color: #1d4ed8;
  padding: 0.45rem 0.65rem;
  border-radius: 10px;
  cursor: pointer;
  font-size: 0.95rem;
}

.action.on {
  background: #1d4ed8;
  color: white;
}

.action.like {
  background: #fff0f6;
  color: #be185d;
}

.action.like.on {
  background: #be185d;
  color: white;
}

.action.bookmark {
  background: #eef2ff;
  color: #1d4ed8;
}

.action.bookmark.on {
  background: #1d4ed8;
  color: white;
}

.empty-state {
  padding: 1rem;
  color: #777;
}
</style>