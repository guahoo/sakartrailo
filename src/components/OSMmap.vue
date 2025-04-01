<template>
  <div class="map-container">
    <div v-if="isLoading" id="loading-indicator">
      <img src="/filler.webp" alt="Loading..." />
      <p>Please be patient, the loading might take a while.</p>

    </div>
    <div id="map"></div>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, ref } from 'vue';
import * as L from 'leaflet';
import 'leaflet/dist/leaflet.css';

export default defineComponent({
  name: 'OSMMap',
  setup() {
    const tracks = ref<any[]>([]); // Ref to store tracks data
    const isLoading = ref(true);

    // Fetch tracks from the API
    const fetchTracks = async () => {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 60000); // 60 секунд

    try {
      console.log('Preparing to fetch tracks...');
      isLoading.value = true;

      const response = await fetch('https://sakartrailo-backend.onrender.com/api/tracks/', {
        signal: controller.signal, // Передаем сигнал аборта
      });

      if (!response.ok) {
        throw new Error('Failed to fetch tracks');
      }

      const data = await response.json();
      console.log('Fetched data:', data);
      tracks.value = data;
    } catch (error) {
      if (error instanceof Error) {
      console.error('Error fetching tracks:', error.message);
      } else {
        console.error('Unknown error:', error);
      }
    } finally {
      clearTimeout(timeoutId); // Очистка таймера
      isLoading.value = false;
    }
  };


    onMounted(async () => {
      console.log('Component mounted, fetching tracks...');
      await fetchTracks(); // Fetch tracks on mount
      

      const mapContainer = document.getElementById('map');
      if (!mapContainer) return;

      const map = L.map(mapContainer).setView([41.6938, 44.8015], 12); 
      
      // L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
      //   maxZoom: 19,
      //   attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
      // }).addTo(map);

      L.tileLayer('https://{s}.basemaps.cartocdn.com/rastertiles/dark_matter/{z}/{x}/{y}{r}.png', {
        attribution: '&copy; <a href="https://carto.com/">CARTO</a>',
        subdomains: ['a', 'b', 'c'],
        maxZoom: 20
       }).addTo(map);
      // If there are tracks, plot them on the map
      if (tracks.value.length > 0) {
        console.log('Tracks available:', tracks.value); // Log the tracks

        tracks.value.forEach((track: any) => {
          if (track.nodes && track.nodes.length > 0) {
            const trackPoints = track.nodes.map((node: any) => [node.lat, node.lon]);

            const labelIcons = {
              yellow: '<img src="/sakartrailo/yellow_marker.svg" width="16">',
              red: '<img src="/sakartrailo/red_marker.svg" width="16">',
              blue: '<img src="/sakartrailo/blue_marker.svg" width="16">',
            };
            const label = track.track_info.label?.toLowerCase() || ""; // Приводим к нижнему регистру для надежности

            let icon = "";
              if (label.includes("yellow")) {
                icon = labelIcons.yellow;
              } else if (label.includes("red")) {
                icon = labelIcons.red;
              } else if (label.includes("blue")) {
                icon = labelIcons.blue;
              }


            const trackPolyline = L.polyline(trackPoints, {
              color: track.track_info.color,
              name: track.track_info.name,
              description: track.track_info.description,
              distance: track.track_info.distance,
              label: track.track_info.label,
              weight: 3,
              lineCap: 'round',
            }).addTo(map);

            trackPolyline.bindPopup(`
             <span style="font-weight: bold;">${track.track_info.name || "No Name"}</span><br>
              ${track.track_info.description ? `<i>${track.track_info.description}</i><br>` : ''}
              Distance: ${(track.track_info.distance ? track.track_info.distance.toFixed(2) : "N/A")} km<br>
               ${icon ? `<span style="display: flex; align-items: center;">
              <span>Marked by: </span>
              ${icon}
            </span>` : ''}
            `);
          } else {
            console.log('No nodes in this track:', track);
          }
        });
      } else {
        console.log('No tracks available.');
      }
    });

    return { isLoading };
  }
});
</script>

<style>
.map-container {
  width: 100vw;
  height: 100vh;
}

#map {
  width: 100%;
  height: 100%;
}
</style>

<style scoped>
#loading-indicator {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000; 
  flex-direction: column;
  align-items: center;
}

#loading-indicator img {
  width: 300px;
  height: 300px;
}

#loading-indicator p {
  font-size: 30px;
  color: #d0d0d0;
  margin-top: 20px;
  white-space: nowrap;
}

@media (max-width: 768px) {
  #loading-indicator p {
    font-size: 24px; 
}
}

@media (max-width: 480px) {
  #loading-indicator p {
    font-size: 18px; 
  }
}
</style>
