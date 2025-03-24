<template>
  <div class="map-container">
    <div v-if="isLoading" id="loading-indicator">
      <img src="/filler.webp" alt="Loading..." />
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
      try {
        console.log('Preparing to fetch tracks...');
        isLoading.value = true;
        const response = await fetch('https://sakartrailo-backend.onrender.com/api/tracks/');
        
        if (!response.ok) {
          throw new Error('Failed to fetch tracks');
        }

        const data = await response.json();
        console.log('Fetched data:', data); // Log the data received
        tracks.value = data; // Store the fetched tracks
      } catch (error) {
        console.error('Error fetching tracks:', error); // Log any errors
      } finally {
        isLoading.value = false; // Set loading to false after fetch completes
      }
    };

    onMounted(async () => {
      console.log('Component mounted, fetching tracks...');
      await fetchTracks(); // Fetch tracks on mount

      const mapContainer = document.getElementById('map');
      if (!mapContainer) return;

      const map = L.map(mapContainer).setView([41.6938, 44.8015], 13);

      L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
        maxZoom: 19,
        attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
      }).addTo(map);

      // If there are tracks, plot them on the map
      if (tracks.value.length > 0) {
        console.log('Tracks available:', tracks.value); // Log the tracks

        tracks.value.forEach((track: any) => {
          if (track.nodes && track.nodes.length > 0) {
            const trackPoints = track.nodes.map((node: any) => [node.lat, node.lon]);

            const labelIcons = {
              yellow: '<img src="/icons/yellow_marker.svg" width="16">',
              red: '<img src="/icons/red_marker.svg" width="16">',
              blue: '<img src="/icons/blue_marker.svg" width="16">',
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
              <b>${track.track_info.name || "No Name"}</b><br>
              ${track.track_info.description ? `<i>${track.track_info.description}</i><br>` : ''}
              Distance: ${(track.track_info.distance ? track.track_info.distance.toFixed(2) : "N/A")} km<br>
              ${icon}
            `);
            map.fitBounds(trackPolyline.getBounds());
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
}

#loading-indicator img {
  width: 500px;
  height: 500px;
}
</style>
