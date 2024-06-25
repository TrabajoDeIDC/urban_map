<script setup>
import { ref, toRaw } from 'vue'
import { GoogleMap, HeatmapLayer } from 'vue3-google-map'
import axios from 'axios'
import config from '../config';

const googleMapsApiKey = config.googleMapsApiKey;

let dataLoaded = ref(false);
const valencia = { lat: 39.479194483821075, lng: -0.373352477968464 };
const currentSection = ref('temp');
const mapData = ref([]);  // Usar ref para reactividad
const selectedDate = ref(new Date().toLocaleDateString('en-CA')); // Inicializar al día actual
const selectedHour = ref(new Date().getHours()); // Inicializar a la hora actual

async function getData() {
  console.log("Trying to fetch " + currentSection.value + " data for day " + selectedDate.value + " at time " + selectedHour.value);

  try {
    const response = await axios.get('http://localhost:8080/data', {
      params: {
        data_type: currentSection.value,
        date: selectedDate.value,
        hour: selectedHour.value
      }
    });
  return response.data;

  } catch (error) {
    console.error('Error fetching data:', error);
  }
}

function handleDataChange(data) {
  currentSection.value = data;
  updateMapData();
}

async function updateMapData() {
  dataLoaded.value = false;
  let dataType = currentSection.value;
  console.log("About to update map data with " + dataType + " data");
  try {
    let data = await getData();
    let newData = [];
    if (data !== null) {
      
      // Data reading
      let locations = [];
      let values = [];
      let maxValue = 0;
      data.forEach(element => {
        locations.push({ "lat": element.lat, "lng": element.lng });
        values.push({ "weight": element.value, "timestamp": element.timestamp});
        if (element.value > maxValue) maxValue = element.value;
      });

      // Weight processing
      values.forEach((value, index) => {
        const relatedValue = value.weight/maxValue;
        const timestamp = new Date(value.timestamp).getMinutes();
        const finalWeight = relatedValue/(60-timestamp);
        newData.push({ location: locations.at(index), weight: finalWeight })
      });
    }false
      
    mapData.value = newData;
    dataLoaded.value = true;
  } catch (error) {
    console.error("Couldn't get data: " + error);
  }
}

updateMapData();
</script>

<template>
  <div>
    <nav class="navbar">
      <ul>
        <li><a href="#section1" @click.prevent="handleDataChange('temp')">Temperature</a></li>
        <li><a href="#section2" @click.prevent="handleDataChange('humidity')">Humidity</a></li>
        <li><a href="#section3" @click.prevent="handleDataChange('air')">Air Quality</a></li>
        <li><a href="#section4" @click.prevent="handleDataChange('noise')">Noise</a></li>
      </ul>
    </nav>
    <div class="controls">
      <label for="date">Date:</label>
      <input type="date" id="date" v-model="selectedDate" @change="updateMapData" />
      <label for="time">Time:</label>
      <input type="range" id="time" v-model="selectedHour" min="0" max="23" @input="updateMapData" />
      <span>{{ toRaw(selectedHour).toString() }}</span>
    </div>
    <div class="map-container">
      <div v-if="!dataLoaded" class="loading-overlay">
        <div class="loading-message">
          LOADING DATA
        </div>
      </div>
      <GoogleMap
        :api-key="googleMapsApiKey"
        :libraries="['visualization']"
        style="width: 100%; height: 100%"
        :center="valencia"
        :zoom="14"
      >
      <HeatmapLayer :options="{ data: toRaw(mapData), radius: 50, dissipating: true }" />
      </GoogleMap>
    </div>
  </div>
</template>

<style scoped>
.map-container {
  width: 1200px;
  height: 800px;
  position: relative;
}

.loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1;
}

.loading-message {
  font-size: 2em;
  color: white;
  background-color: rgba(0, 0, 0, 0.7);
  padding: 20px;
  border-radius: 10px;
}

.navbar {
  background-color: #333;
  overflow: hidden;
}

.navbar ul {
  list-style-type: none;
  margin: 0;
  padding: 0;
  display: flex;
  justify-content: space-around;
}

.navbar li {
  float: left;
}

.navbar li a {
  display: block;
  color: white;
  text-align: center;
  padding: 14px 16px;
  text-decoration: none;
}

.navbar li a:hover {
  background-color: #ddd;
  color: black;
}

.controls {
  margin-top: 10px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.controls label {
  font-weight: bold;
}

.controls input[type="date"] {
  padding: 5px;
  font-size: 16px;
  width: 200px;
}

.controls input[type="range"] {
  width: 800px;
}

</style>
