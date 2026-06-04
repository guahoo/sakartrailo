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
import 'leaflet-textpath';



export default defineComponent({
  name: 'OSMMap',
  setup() {
    const tracks = ref<any[]>([]); 
    const isLoading = ref(true);

    const fetchTracks = async () => {
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), 60000); 
    
    try {
      console.log('Preparing to fetch tracks...');
      isLoading.value = true;

      const response = await fetch('https://sakartrailo-backend.onrender.com/api/tracks/', {
        signal: controller.signal,
      });


      if (!response.ok) {
        throw new Error('Failed to fetch tracks');
      }

      const data = await response.json();
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
      await fetchTracks(); 
      

      const mapContainer = document.getElementById('map');
      if (!mapContainer) return;

      const map = L.map(mapContainer).setView([41.6938, 44.8015], 12); 

      var Stadia_OSMBright = L.tileLayer('https://tiles.stadiamaps.com/tiles/osm_bright/{z}/{x}/{y}{r}.{ext}', {
        minZoom: 0,
        maxZoom: 20,
        attribution: '&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> &copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> &copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        ext: 'png'
      }).addTo(map);


      if (tracks.value.length > 0) {
       
        const groupPolylines: Map<string, any[]> = new Map();
        const groupTrackPoints: Map<string, { name: string; points: [number, number][] }[]> = new Map();

        tracks.value.forEach((track: any) => {
          
          if (track.nodes && track.nodes.length > 0) {
            // const trackPoints = track.nodes.map((node: any) => [node.lat, node.lon]);

            const trackPoints: [number, number][] = track.nodes.map((node: any) => [node.lat, node.lon]);
            const groupId = track.group_id;
            
            if (!groupTrackPoints.has(groupId)) {
              groupTrackPoints.set(groupId, []);
            }

            groupTrackPoints.get(groupId)!.push({
              name: track.track_info.name,
              points: trackPoints
            });

            const initialWeight = 3;  
            const highlightWeight = initialWeight * 2; 
            const initialZIndex = 1;  
            const highlightZIndex = 1000; 


            const labelIcons = {
              yellow: '<img src="/sakartrailo/yellow_marker.svg" width="16">',
              red: '<img src="/sakartrailo/red_marker.svg" width="16">',
              blue: '<img src="/sakartrailo/blue_marker.svg" width="16">',
            };
            const label = track.track_info.label?.toLowerCase() || "";

            let icon = "";
              if (label.includes("yellow")) {
                icon = labelIcons.yellow;
              } else if (label.includes("red")) {
                icon = labelIcons.red;
              } else if (label.includes("blue")) {
                icon = labelIcons.blue;
              }

              const visiblePolyline = L.polyline(trackPoints, {
                  color: track.track_info.color,
                  weight: 3,
                  lineCap: 'round',
                }).addTo(map);

                visiblePolyline.bindPopup(`
                  <strong>${track.track_info.name}</strong><br>
                  ${track.track_info.description}<br>
                  Дистанция: ${track.track_info.distance}
                `);

                const invisibleHitboxPolyline = L.polyline(trackPoints, {
                  color: '#000',
                  weight: 20,
                  opacity: 0,
                  clickable: true,
                  pane: 'shadowPane',
                }).addTo(map);

                if (!groupPolylines.has(groupId)) {
                  groupPolylines.set(groupId, []);
                }
                groupPolylines.get(groupId)!.push(visiblePolyline);

                visiblePolyline.on('popupopen', () => {
                  const group = groupPolylines.get(groupId);
                  group?.forEach(poly => {
                    poly.setStyle({ weight: highlightWeight });
                    poly.bringToFront();
                  });
                });

                visiblePolyline.on('popupclose', () => {
                  const group = groupPolylines.get(groupId);
                  group?.forEach(poly => {
                    poly.setStyle({ weight: initialWeight });
                    poly.bringToBack();
                  });
                });

                invisibleHitboxPolyline.on('click', () => {
                  visiblePolyline.openPopup(); // Открываем popup у видимой линии
                });


        const safeName = track.track_info.name.replace(/\s+/g, '-').toLowerCase();

        const popupContent = `
          <span style="font-weight: bold;">${track.track_info.name || "No Name"}</span><br>
          ${track.track_info.description ? `<i>${track.track_info.description}</i><br>` : ''}
          Distance: ${(track.track_info.distance ? track.track_info.distance.toFixed(2) : "N/A")} km<br>
          ${icon ? `<span style="display: flex; align-items: center;">
            <span>Marked by: </span>
            ${icon}
          </span>` : ''}
          <button id="download-gpx-${safeName}">Download GPX</button>
        `;

        visiblePolyline.bindPopup(popupContent);

        visiblePolyline.on('popupopen', () => {
        const btn = document.getElementById(`download-gpx-${safeName}`);
        if (btn) {
          btn.addEventListener('click', () => {
            const groupId = track.group_id;
            const groupTracks = groupTrackPoints.get(groupId);
            console.log('Group tracks:', groupTracks);
            if (!groupTracks) return;

           
            const orderedTracks =  groupTracks;

            downloadGpxGroup(track.track_info.name, orderedTracks);
          });
        }
      });


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


function downloadGpxGroup(trackName: string, tracks: { name: string; points: [number, number][] }[]) {
  const gpxHeader = `<?xml version="1.0" encoding="UTF-8"?>
<gpx version="1.1" creator="YourApp" xmlns="http://www.topografix.com/GPX/1/1">
  <trk><name>${trackName}</name>`;

  const gpxSegments = tracks.map(track =>
    `<trkseg>
${track.points.map(([lat, lon]) => `<trkpt lat="${lat}" lon="${lon}"></trkpt>`).join('\n')}
</trkseg>`
  ).join('\n');

  const gpxFooter = `</trk></gpx>`;

  const blob = new Blob([`${gpxHeader}\n${gpxSegments}\n${gpxFooter}`], {
    type: 'application/gpx+xml'
  });

  console.log('GPX Blob:', blob);

  const url = URL.createObjectURL(blob);
  const link = document.createElement('a');
  link.href = url;
  link.download = `${trackName || 'track'}.gpx`;
  link.click();
  URL.revokeObjectURL(url);
}
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
