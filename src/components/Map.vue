<script setup>
import { ref, onMounted } from "vue";
import "leaflet/dist/leaflet.css";
import * as L from "leaflet";
import {
  LMap,
  LTileLayer,
  LMarker,
  LPolyline,
  LPolygon,
} from "@vue-leaflet/vue-leaflet";

// Make Leaflet available globally for vue-leaflet
window.L = L;

const zoom = ref(8.5);
const center = ref([41.1408, -8.4041]);
const markers = ref([]);
const directionLines = ref([]);
const mapRef = ref(null);

// Create custom icon for aircraft markers (smaller, radar-style)
const circleIcon = L.divIcon({
  className: "custom-aircraft-icon",
  html: `<svg height="12" width="12">
    <circle cx="6" cy="6" r="4" fill="#00ff00" fill-opacity="0.8" stroke="#00aa00" stroke-width="1"/>
  </svg>`,
  iconSize: [12, 12],
  iconAnchor: [6, 6],
});

// Function to calculate direction line endpoint showing position in 1 minute
function calculateDirectionLine(lat, lon, track, speed, distance = 0.001) {
  if (!track || isNaN(track) || !speed || isNaN(speed)) return null;
  const map = mapRef.value?.leafletObject;
  if (!map) return null;

  // Convert speed from knots to degrees per minute
  // Approximate conversion factors:
  // 1 knot ≈ 0.0003 degrees latitude per minute
  // For longitude, it depends on latitude, but we'll use a simplified approach
  const speedInDegreesPerMinute = speed * 0.0003;

  // Calculate the actual distance the aircraft will cover in 1 minute
  const distanceIn1Minute = speedInDegreesPerMinute / 90; // More precise calculation

  const trackRad = (track * Math.PI) / 180;
  const latRad = (lat * Math.PI) / 180;
  const lonRad = (lon * Math.PI) / 180;

  // Start point - current position
  const startLatRad = latRad;
  const startLonRad = lonRad;

  // End point - position after 1 minute
  const endLatRad = Math.asin(
    Math.sin(startLatRad) * Math.cos(distanceIn1Minute) +
      Math.cos(startLatRad) * Math.sin(distanceIn1Minute) * Math.cos(trackRad)
  );
  const endLonRad =
    startLonRad +
    Math.atan2(
      Math.sin(trackRad) * Math.sin(distanceIn1Minute) * Math.cos(startLatRad),
      Math.cos(distanceIn1Minute) - Math.sin(startLatRad) * Math.sin(endLatRad)
    );

  return [
    [lat, lon], // Current position
    [(endLatRad * 180) / Math.PI, (endLonRad * 180) / Math.PI], // Position in 1 minute
  ];
}

// Fetch and update markers
async function getFlights() {
  try {
    const response = await fetch(
      "https://api.airplanes.live/v2/point/39.304436/-11.082919/250"
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

    directionLines.value = validMarkers
      .map((marker) => {
        if (marker.track) {
          const linePoints = calculateDirectionLine(
            marker.lat,
            marker.lon,
            marker.track,
            marker.gs
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
  }, 3000);
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
      <!--       <l-tile-layer
        url="https://{s}.basemaps.cartocdn.com/dark_nolabels/{z}/{x}/{y}{r}.pngg"
        layer-type="base"
        name="Stadia Maps Basemap" /> -->

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
            html: `
      <div class='flight-number'>
        <div class='line-first'>
          ${marker.flight || marker.hex}${marker.t}
        </div>
        <div class='line-first'>${Math.round(marker.gs)} N${Math.round(
              marker.track
            )} ${marker.alt_baro}
        </div>
        ${marker.squawk}
      </div>`,
            iconSize: [100, 16],
            iconAnchor: [40, 20],
          })
        "
      />
      <!-- lppr - porto airport -->
      <l-polyline
        :lat-lngs="[
          [41.230025, -8.676519],
          [41.263932, -8.6854],
        ]"
        color="yellow"
        weight="2"
      ></l-polyline>
      <!-- Portugal Area -->
      <l-polygon
        :lat-lngs="[
          [41.848995, -8.885654],
          [41.025388, -8.676914],
          [40.216523, -8.91312],
          [40.128379, -8.863681],
          [39.59711, -9.094394],
          [38.696114, -9.514621],
          [38.633922, -9.245456],
          [38.42981, -9.223483],
          [38.494329, -8.921359],
          [38.229435, -8.778537],
          [37.937583, -8.891147],
          [37.915918, -8.803256],
          [37.033137, -9.006503],
          [36.991269, -8.947976],
          [37.114021, -8.546975],
          [36.958355, -7.898781],
          [37.176416, -7.40371],
          [37.55624, -7.521813],
          [37.98611, -7.263634],
          [38.025066, -6.988976],
          [38.206582, -6.96151],
          [38.228161, -7.126305],
          [38.516703, -7.335046],
          [39.000736, -6.96151],
          [39.661438, -7.549279],
          [39.682579, -7.010949],
          [40.062005, -6.87362],
          [40.141838, -7.016442],
          [40.238352, -6.983483],
          [40.343103, -6.802209],
          [40.880639, -6.816358],
          [41.042417, -6.917981],
          [41.570592, -6.19563],
          [41.665045, -6.453809],
          [41.943487, -6.555432],
          [41.954938, -7.165005],
          [41.81794, -7.41769],
          [41.834313, -8.167507],
          [42.036588, -8.082363],
          [42.164974, -8.186733],
        ]"
        color="white"
        :fill="false"
        weight="1"
      ></l-polygon>
      <!-- NORTH SECTORS -->
      <l-polygon
        :lat-lngs="[
          [41.5203, -8.5037],
          [41.4909, -8.0902],
          [41.4849, -8.0335],
          [41.4202, -6.3719],
          [41.2859, -6.1648],
          [41.2853, -6.1708],
          [40.2358, -6.4906],
          [39.23, -8.01],
          [38.54, -10.0],
          [39.2055, -9.4705],
          [40.1159, -9.2633],
          [40.2256, -9.2205],
          [40.3856, -9.15],
          [40.494, -9.1111],
          [41.5203, -8.5037],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
      ></l-polygon>

      <!-- CENTRE SECTORS -->
      <l-polygon
        :lat-lngs="[
          [40.2358, -6.4906],
          [39.07, -7.01],
          [39.0632, -7.0142],
          [38.5008, -7.0443],
          [38.4917, -7.0505],
          [38.1812, -7.1047],
          [38.1758, -7.1034],
          [37.5958, -7.1348],
          [38.0, -9.0],
          [38.0, -9.12],
          [38.0, -9.27],
          [38.0, -9.28],
          [38.0, -10.0],
          [38.54, -10.0],
          [39.23, -8.01],
          [40.2358, -6.4906],
        ]"
        color="#ffff00"
        :fill="true"
        fill-color="rgba(255, 255, 0, 0.1)"
        :weight="1"
      ></l-polygon>

      <!-- SOUTH SECTORS -->
      <l-polygon
        :lat-lngs="[
          [37.5958, -7.1348],
          [37.5928, -7.1409],
          [37.1016, -7.234],
          [37.0744, -7.23],
          [36.4016, -7.2311],
          [35.58, -7.23],
          [35.58, -10.44],
          [38.0, -9.28],
          [38.0, -9.27],
          [38.0, -9.12],
          [38.0, -9.0],
          [37.5958, -7.1348],
        ]"
        color="#ff00ff"
        :fill="true"
        fill-color="rgba(255, 0, 255, 0.1)"
        :weight="1"
      ></l-polygon>

      <!-- WEST SECTOR -->
      <l-polygon
        :lat-lngs="[
          [42.0, -15.0],
          [43.0, -13.0],
          [42.0126, -10.0405],
          [41.5222, -8.5536],
          [41.5308, -8.5015],
          [40.494, -9.1111],
          [40.3856, -9.1505],
          [40.2256, -9.2205],
          [40.1159, -9.2633],
          [39.2055, -9.4705],
          [38.54, -10.0],
          [38.0, -10.0],
          [35.58, -10.44],
          [35.58, -12.0],
          [36.0323, -12.3329],
          [36.4621, -13.4031],
          [36.3, -15.0],
          [39.3, -15.0],
          [42.0, -15.0],
        ]"
        color="#00ffff"
        :fill="true"
        fill-color="rgba(0, 255, 255, 0.1)"
        :weight="1"
      ></l-polygon>
      <!-- PORTO TMA Main Sector -->
      <l-polygon
        :lat-lngs="[
          [41.5335, -9.0157],
          [41.5221, -8.5126],
          // Along Portuguese/Spanish border (approximated)
          [41.5145, -8.0],
          [41.5145, -7.2708],
          [41.0213, -7.5854],
          // 35NM arc centered at 41.1623, -8.4116 (approximated)
          [41.1214, -9.272],
          [41.5335, -9.0157],
        ]"
        color="#ff6600"
        :fill="true"
        fill-color="rgba(255, 102, 0, 0.15)"
        :weight="2"
      ></l-polygon>

      <!-- ÁREA N - 8.5NM RADIUS CENTRED ON 412951N 0070538W -->
      <l-circle
        :lat-lng="[41.4975, -7.0939]"
        :radius="15742"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="2"
        dash-array="5,5"
      ></l-circle>

      <!-- ÁREA N1 -->
      <l-polygon
        :lat-lngs="[
          [42.0764, -8.5047],
          [41.8431, -8.5047],
          [41.8725, -8.8572],
          [42.0764, -8.5047],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N2 -->
      <l-polygon
        :lat-lngs="[
          [42.0764, -8.5047],
          [41.8153, -8.1633],
          [41.7067, -8.1633],
          [41.8431, -8.5047],
          [42.0764, -8.5047],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N3 -->
      <l-polygon
        :lat-lngs="[
          [41.8311, -7.5628],
          [41.6397, -7.5628],
          [41.6397, -8.0747],
          [41.7067, -8.1633],
          [41.8153, -8.1633],
          [41.8311, -7.5628],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N4 -->
      <l-polygon
        :lat-lngs="[
          [41.9806, -7.1694],
          [41.6397, -7.1694],
          [41.6397, -7.5628],
          [41.8311, -7.5628],
          [41.9806, -7.1694],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N5 -->
      <l-polygon
        :lat-lngs="[
          [41.9878, -6.7933],
          [41.6397, -6.7933],
          [41.6397, -7.1694],
          [41.9806, -7.1694],
          [41.9878, -6.7933],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N6 -->
      <l-polygon
        :lat-lngs="[
          [41.9878, -6.7933],
          [41.6397, -6.2628],
          [41.6397, -6.7933],
          [41.9878, -6.7933],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N7 -->
      <l-polygon
        :lat-lngs="[
          [41.6397, -8.0747],
          [41.6397, -7.5628],
          [41.3578, -7.5628],
          [41.3578, -7.9239],
          [41.4836, -7.9661],
          [41.6397, -8.0747],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N8 -->
      <l-polygon
        :lat-lngs="[
          [41.6397, -7.5628],
          [41.6397, -7.1694],
          [41.3578, -7.1694],
          [41.3578, -7.5628],
          [41.6397, -7.5628],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N9 -->
      <l-polygon
        :lat-lngs="[
          [41.6397, -7.1694],
          [41.6397, -6.7933],
          [41.3578, -6.7933],
          [41.3578, -7.1694],
          [41.6397, -7.1694],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N10 -->
      <l-polygon
        :lat-lngs="[
          [41.6397, -6.7933],
          [41.6397, -6.2628],
          [41.3578, -6.3953],
          [41.3578, -6.7933],
          [41.6397, -6.7933],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N11 -->
      <l-polygon
        :lat-lngs="[
          [41.3578, -7.9239],
          [41.3578, -7.5628],
          [41.0694, -7.5628],
          [41.0694, -7.9678],
          [41.1903, -7.9311],
          [41.3578, -7.9239],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N12 -->
      <l-polygon
        :lat-lngs="[
          [41.3578, -7.5628],
          [41.3578, -7.1694],
          [41.0694, -7.1694],
          [41.0694, -7.5628],
          [41.3578, -7.5628],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N13 -->
      <l-polygon
        :lat-lngs="[
          [41.3578, -7.1694],
          [41.3578, -6.7933],
          [41.0694, -6.7933],
          [41.0694, -7.1694],
          [41.3578, -7.1694],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N14 -->
      <l-polygon
        :lat-lngs="[
          [41.3578, -6.7933],
          [41.3578, -6.3953],
          [41.0694, -6.7814],
          [41.0694, -6.7933],
          [41.3578, -6.7933],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N15 -->
      <l-polygon
        :lat-lngs="[
          [41.0694, -7.9678],
          [41.0694, -7.5628],
          [40.8381, -7.4747],
          [40.8381, -7.8172],
          [40.9736, -7.8269],
          [41.0694, -7.9678],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N16 -->
      <l-polygon
        :lat-lngs="[
          [41.0694, -7.5628],
          [41.0694, -7.1694],
          [40.6953, -7.1694],
          [40.7553, -7.4747],
          [40.8381, -7.4747],
          [41.0694, -7.5628],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA N17 -->
      <l-polygon
        :lat-lngs="[
          [41.0694, -7.1694],
          [41.0694, -6.7933],
          [41.0694, -6.7814],
          [40.7203, -6.8175],
          [40.6953, -7.1694],
          [41.0694, -7.1694],
        ]"
        color="#ff0000"
        :fill="true"
        fill-color="rgba(255, 0, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L - 7NM RADIUS CENTRED ON 400524N 0081043W -->
      <l-circle
        :lat-lng="[40.09, -8.1786]"
        :radius="12964"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="2"
        dash-array="5,5"
      ></l-circle>

      <!-- ÁREA L1 -->
      <l-polygon
        :lat-lngs="[
          [41.0694, -7.9678],
          [40.9736, -7.8269],
          [40.8381, -7.8172],
          [40.8381, -8.1786],
          [40.9625, -8.0214],
          [41.0694, -7.9678],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L2 -->
      <l-polygon
        :lat-lngs="[
          [40.8381, -8.1786],
          [40.6317, -8.1786],
          [40.6317, -8.5014],
          [40.7067, -8.5014],
          [40.7528, -8.3278],
          [40.8381, -8.1786],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L3 -->
      <l-polygon
        :lat-lngs="[
          [40.8381, -8.1786],
          [40.8381, -7.8172],
          [40.6317, -7.8172],
          [40.6317, -8.1786],
          [40.8381, -8.1786],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L4 -->
      <l-polygon
        :lat-lngs="[
          [40.8381, -7.8172],
          [40.8381, -7.4747],
          [40.7553, -7.4747],
          [40.6317, -7.4747],
          [40.6317, -7.8172],
          [40.8381, -7.8172],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L5 -->
      <l-polygon
        :lat-lngs="[
          [40.6317, -8.5014],
          [40.3667, -8.5014],
          [40.3667, -8.7867],
          [40.6317, -8.5014],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L6 -->
      <l-polygon
        :lat-lngs="[
          [40.6317, -8.5014],
          [40.6317, -8.1786],
          [40.3667, -8.1786],
          [40.3667, -8.5014],
          [40.6317, -8.5014],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L7 -->
      <l-polygon
        :lat-lngs="[
          [40.6317, -8.1786],
          [40.6317, -7.8172],
          [40.3667, -7.8172],
          [40.3667, -8.1786],
          [40.6317, -8.1786],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L8 -->
      <l-polygon
        :lat-lngs="[
          [40.6317, -7.8172],
          [40.6317, -7.4747],
          [40.3667, -7.3222],
          [40.3667, -7.8172],
          [40.6317, -7.8172],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L9 -->
      <l-polygon
        :lat-lngs="[
          [40.3667, -8.7867],
          [40.3667, -8.5014],
          [40.09, -8.6603],
          [40.09, -8.8856],
          [40.3667, -8.7867],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L10 -->
      <l-polygon
        :lat-lngs="[
          [40.3667, -8.5014],
          [40.3667, -8.1786],
          [40.09, -8.1786],
          [40.09, -8.6603],
          [40.3667, -8.5014],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L11 -->
      <l-polygon
        :lat-lngs="[
          [40.3667, -8.1786],
          [40.3667, -7.8172],
          [40.09, -7.8172],
          [40.09, -8.1786],
          [40.3667, -8.1786],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L12 -->
      <l-polygon
        :lat-lngs="[
          [40.3667, -7.8172],
          [40.3667, -7.3222],
          [40.09, -7.2444],
          [40.09, -7.8172],
          [40.3667, -7.8172],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L13 -->
      <l-polygon
        :lat-lngs="[
          [40.09, -8.6603],
          [40.09, -8.1786],
          [39.8097, -8.1786],
          [39.8097, -8.5275],
          [39.8097, -8.6225],
          [40.09, -8.6603],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L14 -->
      <l-polygon
        :lat-lngs="[
          [40.09, -8.1786],
          [40.09, -7.8172],
          [39.8097, -7.8172],
          [39.8097, -8.1786],
          [40.09, -8.1786],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L15 -->
      <l-polygon
        :lat-lngs="[
          [40.09, -7.8172],
          [40.09, -7.2444],
          [39.8097, -7.3222],
          [39.8097, -7.8172],
          [40.09, -7.8172],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L16 -->
      <l-polygon
        :lat-lngs="[
          [39.8097, -8.6225],
          [39.8097, -8.5275],
          [39.5533, -8.5275],
          [39.5533, -8.7556],
          [39.7236, -8.6181],
          [39.8097, -8.6225],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L17 -->
      <l-polygon
        :lat-lngs="[
          [39.8097, -8.5275],
          [39.8097, -8.1786],
          [39.5533, -8.1786],
          [39.5533, -8.5275],
          [39.8097, -8.5275],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L18 -->
      <l-polygon
        :lat-lngs="[
          [39.8097, -8.1786],
          [39.8097, -7.8172],
          [39.6761, -7.8172],
          [39.7289, -7.9692],
          [39.5533, -8.0733],
          [39.5533, -8.1786],
          [39.8097, -8.1786],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L19 -->
      <l-polygon
        :lat-lngs="[
          [39.8097, -7.8172],
          [39.8097, -7.3222],
          [39.6494, -7.3492],
          [39.5911, -7.5022],
          [39.605, -7.6083],
          [39.6761, -7.8172],
          [39.8097, -7.8172],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L20 -->
      <l-polygon
        :lat-lngs="[
          [39.5533, -8.5275],
          [39.5533, -8.1786],
          [39.5533, -8.0733],
          [39.3069, -8.2181],
          [39.3069, -8.5275],
          [39.5533, -8.5275],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L21 -->
      <l-polygon
        :lat-lngs="[
          [39.605, -7.6083],
          [39.5911, -7.5022],
          [39.4844, -7.3694],
          [39.3069, -7.3667],
          [39.3069, -7.7853],
          [39.605, -7.6083],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L22 -->
      <l-polygon
        :lat-lngs="[
          [39.3069, -7.7853],
          [39.3069, -7.3667],
          [39.0644, -7.4786],
          [39.0644, -7.9339],
          [39.3069, -7.7853],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA L23 -->
      <l-polygon
        :lat-lngs="[
          [39.0644, -7.9339],
          [39.0644, -7.4786],
          [38.7969, -7.8172],
          [38.7969, -8.1736],
          [38.8333, -8.1667],
          [39.0517, -8.1189],
          [38.9983, -7.9694],
          [39.0644, -7.9339],
        ]"
        color="#00ff00"
        :fill="true"
        fill-color="rgba(0, 255, 0, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA S - 5NM RADIUS CENTRED ON 371854N 0083457W -->
      <l-circle
        :lat-lng="[37.315, -8.5825]"
        :radius="9260"
        color="#0000ff"
        :fill="true"
        fill-color="rgba(0, 0, 255, 0.1)"
        :weight="2"
        dash-array="5,5"
      ></l-circle>

      <!-- ÁREA S1 -->
      <l-polygon
        :lat-lngs="[
          [38.7969, -8.1736],
          [38.7969, -7.8172],
          [38.6006, -7.8172],
          [38.6006, -7.9403],
          [38.5486, -7.9836],
          [38.5486, -8.0711],
          [38.5467, -8.3047],
          [38.7969, -8.1736],
        ]"
        color="#0000ff"
        :fill="true"
        fill-color="rgba(0, 0, 255, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA S2 -->
      <l-polygon
        :lat-lngs="[
          [38.7969, -7.8172],
          [38.7, -7.4786],
          [38.5525, -7.2853],
          [38.5525, -7.8064],
          [38.6006, -7.8172],
          [38.7969, -7.8172],
        ]"
        color="#0000ff"
        :fill="true"
        fill-color="rgba(0, 0, 255, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA S3 -->
      <l-polygon
        :lat-lngs="[
          [38.5467, -8.3047],
          [38.5486, -8.0711],
          [38.3486, -8.2678],
          [38.2003, -8.2058],
          [38.2003, -8.4],
          [38.2003, -8.6736],
          [38.3667, -8.4],
          [38.5467, -8.3047],
        ]"
        color="#0000ff"
        :fill="true"
        fill-color="rgba(0, 0, 255, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA S4 -->
      <l-polygon
        :lat-lngs="[
          [38.2003, -8.6736],
          [38.2003, -8.4],
          [37.8986, -8.4],
          [37.8986, -8.8064],
          [38.1342, -8.7914],
          [38.2003, -8.6736],
        ]"
        color="#0000ff"
        :fill="true"
        fill-color="rgba(0, 0, 255, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA S5 -->
      <l-polygon
        :lat-lngs="[
          [38.2003, -8.4],
          [38.2003, -8.2061],
          [37.8986, -8.0794],
          [37.8986, -8.4],
          [38.2003, -8.4],
        ]"
        color="#0000ff"
        :fill="true"
        fill-color="rgba(0, 0, 255, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- ÁREA S6 -->
      <l-polygon
        :lat-lngs="[
          [37.8986, -8.8064],
          [37.8986, -8.4],
          [37.5975, -8.4],
          [37.5975, -8.5825],
          [37.5975, -8.8064],
          [37.8986, -8.8064],
        ]"
        color="#0000ff"
        :fill="true"
        fill-color="rgba(0, 0, 255, 0.1)"
        :weight="1"
        dash-array="5,5"
      ></l-polygon>

      <!-- 35NM Circle around PRT VOR/DME -->
      <l-circle :lat-lng="[41.1449, -8.4108]" :radius="35000 * 1.852 * 1000">
        color="#ffffff" :fill="false" :weight="1" dash-array="5,10" ></l-circle
      >

      <!-- Legend/Controls (optional) -->
      <l-control position="topright">
        <div class="map-legend">
          <h4>PORTO TMA Sectors</h4>
          <div>
            <span style="background: rgba(255, 102, 0, 0.15)"></span> Main TMA
            (FL245-SFC)
          </div>
          <div>
            <span
              style="
                background: rgba(0, 204, 255, 0.1);
                border: 1px dashed #00ccff;
              "
            ></span>
            Vectoring Area (FL115-300M)
          </div>
          <div><span style="border: 1px dashed #fff"></span> 35NM Circle</div>
        </div>
      </l-control>
      <!-- MADEIRA SECTOR -->
      <l-polygon
        :lat-lngs="[
          [35.58, -12.0],
          [32.25, -14.6333],
          [33.0407, -16.213], // Arc points would need to be calculated
          [34.25, -17.7667],
          [36.3, -15.0],
          [36.4621, -13.4031],
          [36.0323, -12.3329],
          [35.58, -12.0],
        ]"
        color="#ff9900"
        :fill="true"
        fill-color="rgba(255, 153, 0, 0.1)"
        :weight="1"
      ></l-polygon>
    </l-map>
  </main>
</template>

<style scoped>
@import url("https://fonts.googleapis.com/css2?family=Space+Mono:ital,wght@0,400;0,700;1,400;1,700&display=swap");

html,
body {
  margin: 0;
  padding: 0;
  height: 100%;
  font-family: "Space Mono", monospace, -apple-system, BlinkMacSystemFont,
    "Segoe UI", Roboto, sans-serif;
}

main {
  height: 100dvh;
  width: 100dvw;
  position: relative;
}

.flight-number {
  max-width: 500px !important;
  overflow: hidden;
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

.line-first {
  display: flex;
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
/* Pure black map background */
:deep(.leaflet-container) {
  background: #000000 !important;
}

/* Hide tile grid (if any remains) */
:deep(.leaflet-tile) {
  opacity: 0 !important;
}

/* Flight label styling */
:deep(.flight-label) {
  background: none !important;
  border: none !important;
}

:deep(.flight-number) {
  color: #00ff00;
  font-family: "Space Mono", monospace;
  font-size: 12px;
  line-height: 1.4;
  text-align: left;
  white-space: nowrap;
}

/* Make the controls use Space Mono */
.controls {
  font-family: "Space Mono", monospace;
}

.update-btn {
  font-family: "Space Mono", monospace;
}

.aircraft-count {
  font-family: "Space Mono", monospace;
}
</style>
