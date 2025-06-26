<script setup>
import { ref, onMounted } from "vue";
import "leaflet/dist/leaflet.css";
import * as L from "leaflet";
import {
  LMap,
  LTileLayer,
  LMarker,
  LPopup,
  LPolyline,
} from "@vue-leaflet/vue-leaflet";

// Make Leaflet available globally for vue-leaflet
window.L = L;

const zoom = ref(9);
const center = ref([41.2, -8.3]);
const markers = ref([]);
const directionLines = ref([]);
const mapRef = ref(null);
let size = -0.0002;

// Create custom icon for aircraft markers (smaller, radar-style)
const circleIcon = L.divIcon({
  className: "custom-aircraft-icon",
  html: `<svg height="12" width="12">
    <circle cx="6" cy="6" r="4" fill="#00ff00" fill-opacity="0.8" stroke="#00aa00" stroke-width="1"/>
  </svg>`,
  iconSize: [12, 12],
  iconAnchor: [6, 6],
});

// Function to calculate direction line endpoint (shorter, starting outside circle)
function calculateDirectionLine(lat, lon, track, distance = 0.001) {
  if (!track || isNaN(track)) return null;

  const trackRad = (track * Math.PI) / 180;
  const latRad = (lat * Math.PI) / 180;
  const lonRad = (lon * Math.PI) / 180;

  // Start point - slightly outside the circle (offset from aircraft position)
  const startDistance = 0.00001; // Small offset to start outside the circle
  const startLatRad = Math.asin(
    Math.sin(latRad) * Math.cos(startDistance) +
      Math.cos(latRad) * Math.sin(startDistance) * Math.cos(trackRad)
  );
  const startLonRad =
    lonRad +
    Math.atan2(
      Math.sin(trackRad) * Math.sin(startDistance) * Math.cos(latRad),
      Math.cos(startDistance) - Math.sin(latRad) * Math.sin(startLatRad)
    );
  console.log(size);

  // End point - short line extending from start point
  const endLatRad = Math.asin(
    Math.sin(startLatRad) * Math.cos(distance - size) +
      Math.cos(startLatRad) * Math.sin(distance - size) * Math.cos(trackRad)
  );
  const endLonRad =
    startLonRad +
    Math.atan2(
      Math.sin(trackRad) * Math.sin(distance - size) * Math.cos(startLatRad),
      Math.cos(distance - size) - Math.sin(startLatRad) * Math.sin(endLatRad)
    );

  return [
    [(startLatRad * 180) / Math.PI, (startLonRad * 180) / Math.PI],
    [(endLatRad * 180) / Math.PI, (endLonRad * 180) / Math.PI],
  ];
}

// Function to create wind rose SVG
function createWindRose(track) {
  const size = 120;
  const center = size / 2;
  const radius = 45;

  // Cardinal directions
  const directions = [
    { name: "N", angle: 0 },
    { name: "NE", angle: 45 },
    { name: "E", angle: 90 },
    { name: "SE", angle: 135 },
    { name: "S", angle: 180 },
    { name: "SW", angle: 225 },
    { name: "W", angle: 270 },
    { name: "NW", angle: 315 },
  ];

  let svg = `<svg width="${size}" height="${size}" viewBox="0 0 ${size} ${size}">`;

  // Background circle
  svg += `<circle cx="${center}" cy="${center}" r="${radius}" fill="rgba(255,255,255,0.1)" stroke="rgba(255,255,255,0.3)" stroke-width="2"/>`;

  // Direction lines and labels
  directions.forEach((dir) => {
    const angle = ((dir.angle - 90) * Math.PI) / 180; // Adjust for SVG coordinate system
    const x1 = center + (radius - 15) * Math.cos(angle);
    const y1 = center + (radius - 15) * Math.sin(angle);
    const x2 = center + radius * Math.cos(angle);
    const y2 = center + radius * Math.sin(angle);
    const textX = center + (radius + 12) * Math.cos(angle);
    const textY = center + (radius + 12) * Math.sin(angle);

    // Direction line
    svg += `<line x1="${x1}" y1="${y1}" x2="${x2}" y2="${y2}" stroke="rgba(255,255,255,0.5)" stroke-width="1"/>`;

    // Direction label
    svg += `<text x="${textX}" y="${textY}" text-anchor="middle" dominant-baseline="middle" fill="white" font-size="12" font-weight="bold">${dir.name}</text>`;
  });

  // Aircraft direction arrow (if track is available)
  if (track !== null && !isNaN(track)) {
    const trackAngle = ((track - 90) * Math.PI) / 180;
    const arrowLength = radius - 5;
    const arrowX = center + arrowLength * Math.cos(trackAngle);
    const arrowY = center + arrowLength * Math.sin(trackAngle);

    // Arrow line
    svg += `<line x1="${center}" y1="${center}" x2="${arrowX}" y2="${arrowY}" stroke="red" stroke-width="3"/>`;

    // Arrow head
    const headLength = 8;
    const headAngle = 0.5;
    const head1X = arrowX - headLength * Math.cos(trackAngle - headAngle);
    const head1Y = arrowY - headLength * Math.sin(trackAngle - headAngle);
    const head2X = arrowX - headLength * Math.cos(trackAngle + headAngle);
    const head2Y = arrowY - headLength * Math.sin(trackAngle + headAngle);

    svg += `<polygon points="${arrowX},${arrowY} ${head1X},${head1Y} ${head2X},${head2Y}" fill="red"/>`;

    // Track value in center
    svg += `<text x="${center}" y="${
      center - 5
    }" text-anchor="middle" dominant-baseline="middle" fill="red" font-size="14" font-weight="bold">${Math.round(
      track
    )}°</text>`;
    svg += `<text x="${center}" y="${
      center + 10
    }" text-anchor="middle" dominant-baseline="middle" fill="white" font-size="10">TRACK</text>`;
  }

  svg += "</svg>";
  return svg;
}

// Fetch and update markers
async function getFlights() {
  try {
    const response = await fetch(
      "https://corsproxy.io/?" +
        encodeURIComponent("https://api.adsb.lol/v2/lat/41.2/lon/-8.3/dist/250")
    );

    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }

    const data = await response.json();

    // Filter out markers without valid coordinates
    const validMarkers =
      data.ac?.filter(
        (marker) =>
          marker.lat && marker.lon && !isNaN(marker.lat) && !isNaN(marker.lon)
      ) || [];

    markers.value = validMarkers;

    // Calculate direction lines
    directionLines.value = validMarkers
      .map((marker) => {
        if (marker.track) {
          const linePoints = calculateDirectionLine(
            marker.lat,
            marker.lon,
            marker.track
          );
          if (linePoints) {
            return {
              hex: marker.hex,
              line: linePoints,
            };
          }
        }
        return null;
      })
      .filter((line) => line !== null);

    console.log(`Updated ${validMarkers.length} aircraft markers`);
  } catch (err) {
    console.error("Failed to fetch aircraft data:", err);
  }
}

function updateMap() {
  getFlights();
}

// Set up periodic updates
function startPeriodicUpdates() {
  setInterval(() => {
    getFlights();
  }, 10000);
}

onMounted(() => {
  getFlights();
  startPeriodicUpdates();
});
</script>

<template>
  <main>
    <div class="controls">
      <button @click="updateMap" class="update-btn">
        Update Aircraft Positions
      </button>
      <span class="aircraft-count"> Aircraft: {{ markers.length }} </span>
    </div>

    <l-map
      ref="mapRef"
      v-model:zoom="zoom"
      v-model:center="center"
      :useGlobalLeaflet="false"
      style="height: calc(100% - 60px); width: 100%"
    >
      <l-tile-layer
        url="https://tiles.stadiamaps.com/tiles/alidade_smooth_dark/{z}/{x}/{y}{r}.png"
        layer-type="base"
        name="Stadia Maps Basemap"
      />

      <!-- Direction lines (radar-style) -->
      <l-polyline
        v-for="line in directionLines"
        :key="`line-${line.hex}`"
        :lat-lngs="line.line"
        :color="'#00ff00'"
        :weight="2"
        :opacity="1"
      />

      <!-- Aircraft markers -->
      <l-marker
        v-for="marker in markers"
        :key="marker.hex"
        :icon="circleIcon"
        :lat-lng="[marker.lat, marker.lon]"
      >
        <l-popup>
          <div class="popup-content">
            <!-- Flight info header -->
            <div class="flight-header">
              <strong>{{ marker.flight || "Unknown Flight" }}</strong>
            </div>

            <!-- Wind Rose -->
            <div
              class="wind-rose-container"
              v-html="createWindRose(marker.track)"
            ></div>

            <!-- Aircraft details -->
            <div class="aircraft-details">
              <div class="detail-row">
                <span class="label">Altitude:</span>
                <span class="value">{{ marker.alt_baro || "N/A" }} ft</span>
              </div>
              <div class="detail-row">
                <span class="label">Speed:</span>
                <span class="value"
                  >{{ Math.round(marker.gs) || "N/A" }} kts</span
                >
              </div>
              <div class="detail-row">
                <span class="label">Track:</span>
                <span class="value">{{
                  marker.track ? Math.round(marker.track) + "°" : "N/A"
                }}</span>
              </div>
              <div class="detail-row">
                <span class="label">Squawk:</span>
                <span class="value">{{ marker.squawk || "0000" }}</span>
              </div>
              <div class="detail-row">
                <span class="label">Registration:</span>
                <span class="value">{{ marker.r || "Unknown" }}</span>
              </div>
              <div class="detail-row">
                <span class="label">Type:</span>
                <span class="value">{{ marker.t || "Unknown" }}</span>
              </div>
              <div class="detail-row">
                <span class="label">Hex:</span>
                <span class="value">{{ marker.hex || "N/A" }}</span>
              </div>
            </div>
          </div>
        </l-popup>
      </l-marker>

      <!-- Flight number labels (closer to aircraft) -->
      <l-marker
        v-for="marker in markers"
        :key="`label-${marker.hex}`"
        :lat-lng="[
          marker.lat + 0.01 - zoom / 900,
          marker.lon - 0.03 + zoom / 400,
        ]"
        :icon="
          L.divIcon({
            className: 'flight-label',
            html: `<div class='flight-number'>${marker.flight || marker.hex}${
              marker.t
            }</br>${marker.alt_baro}</br>${Math.round(marker.track)}</div>`,
            iconSize: [80, 16],
            iconAnchor: [40, 20],
          })
        "
      />
    </l-map>
  </main>
</template>

<style scoped>
html,
body {
  margin: 0;
  padding: 0;
  height: 100%;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
}

main {
  height: 100dvh;
  width: 100dvw;
  position: relative;
}

.controls {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1000;
  display: flex;
  align-items: center;
  gap: 15px;
  background: rgba(0, 0, 0, 0.8);
  padding: 10px 15px;
  border-radius: 8px;
  backdrop-filter: blur(10px);
}

.update-btn {
  background: #dc3545;
  color: white;
  border: none;
  padding: 8px 16px;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 500;
  transition: background-color 0.2s;
}

.update-btn:hover {
  background: #c82333;
}

.aircraft-count {
  color: white;
  font-size: 14px;
  font-weight: 500;
}

.popup-content {
  min-width: 250px;
  text-align: center;
}

.flight-header {
  margin-bottom: 15px;
  padding-bottom: 10px;
  border-bottom: 1px solid rgba(255, 255, 255, 0.2);
  font-size: 16px;
}

.wind-rose-container {
  margin: 15px 0;
  display: flex;
  justify-content: center;
}

.aircraft-details {
  margin-top: 15px;
  text-align: left;
  border-top: 1px solid rgba(255, 255, 255, 0.2);
  padding-top: 10px;
}

.detail-row {
  display: flex;
  justify-content: space-between;
  margin: 4px 0;
  font-size: 13px;
}

.label {
  font-weight: 600;
  color: #bbb;
}

.value {
  font-weight: 500;
  color: white;
}

/* Custom icon styling */
:deep(.custom-aircraft-icon) {
  background: none !important;
  border: none !important;
}

/* Flight label styling */
:deep(.flight-label) {
  background: none !important;
  border: none !important;
}

:deep(.flight-number) {
  color: #00ff00;
  font-family: "Courier New", monospace;
}

/* Map styling adjustments */
:deep(.leaflet-popup-content-wrapper) {
  background: #2c3e50;
  color: white;
  border-radius: 12px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.4);
}

:deep(.leaflet-popup-tip) {
  background: #2c3e50;
}

:deep(.leaflet-popup-content) {
  margin: 15px 18px;
}
</style>
