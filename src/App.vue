<template>
  <div>
    <input v-model="searchQuery1" placeholder="Enter city 1" />
    <input v-model="searchQuery2" placeholder="Enter city 2" />
    <button @click="search">Search</button>
    <l-map :zoom="zoom" :center="center" style="height: 600px">
      <l-tile-layer :url="tileLayerUrl"></l-tile-layer>
      <l-marker
        v-for="(marker, index) in markers"
        :key="index"
        :lat-lng="marker.position"
      ></l-marker>
      <l-polyline
        :lat-lngs="path"
        :editable="true"
        @editable:vertex:dragend="updatePath"
      ></l-polyline>
    </l-map>
    <div v-if="path">
      <p>Distance: {{ calculateDistance().toFixed(2) / 1000 }} Km</p>
      <p>Périmètre: {{ calculatePerimeter().toFixed(2) / 1000 }} Km</p>
      <!-- <p>Surface: {{ calculateArea().toFixed(2) / 1000 }} mètre carré</p> -->
    </div>
    <button @click="zoomIn">Zoom In</button>
    <button @click="zoomOut">Zoom Out</button>
  </div>
</template>

<script>
import L from "leaflet";
import { LMap, LTileLayer, LMarker, LPolyline } from "vue2-leaflet";
import "leaflet/dist/leaflet.css";

export default {
  components: {
    LMap,
    LTileLayer,
    LMarker,
    LPolyline,
  },
  data() {
    return {
      searchQuery1: "",
      searchQuery2: "",
      center: L.latLng(0, 0),
      zoom: 1,
      markers: [],
      path: [],
      tileLayerUrl: "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",
    };
  },
  methods: {
    search() {
      this.center = L.latLng(0, 0); // Reset the map center
      this.markers = []; // Clear previous markers

      const searchPromises = [
        fetch(
          `https://nominatim.openstreetmap.org/search.php?q=${this.searchQuery1}&format=json`
        ),
        fetch(
          `https://nominatim.openstreetmap.org/search.php?q=${this.searchQuery2}&format=json`
        ),
      ];

      Promise.all(searchPromises)
        .then((responses) =>
          Promise.all(responses.map((response) => response.json()))
        )
        .then((data) => {
          const city1 = data[0][0];
          const city2 = data[1][0];

          if (city1 && city2) {
            this.center = L.latLng(city1.lat, city1.lon);
            this.markers.push({ position: this.center });

            const city2LatLng = L.latLng(city2.lat, city2.lon);
            this.markers.push({ position: city2LatLng });

            this.zoom = 10; // Zoom in to show the markers

            this.path = [this.center, city2LatLng];
          }
        });
    },
    zoomIn() {
      this.zoom++;
    },
    zoomOut() {
      if (this.zoom > 0) {
        this.zoom--;
      }
    },
    updatePath(event) {
      this.path = event.target.getLatLngs();
    },
    calculateDistance() {
      if (this.path.length < 2) return 0;
      let distance = 0;
      for (let i = 1; i < this.path.length; i++) {
        const prev = this.path[i - 1];
        const curr = this.path[i];
        distance += prev.distanceTo(curr);
      }
      return distance;
    },
    calculatePerimeter() {
      if (this.path.length < 2) return 0;
      let perimeter = 0;
      for (let i = 1; i < this.path.length; i++) {
        const prev = this.path[i - 1];
        const curr = this.path[i];
        perimeter += prev.distanceTo(curr);
      }
      perimeter += this.path[this.path.length - 1].distanceTo(this.path[0]);
      return perimeter;
    },
    calculateArea() {
      if (this.path.length < 3) return 0;
      const polygon = L.polygon(this.path);
      return polygon.getArea();
    },
  },
};
</script>
