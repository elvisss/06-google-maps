<template>
  <div class="map">
		<h1>Mapa</h1>
		<hr>
		<div ref="map" id="map"></div>
  </div>
</template>

<script lang="ts">
/* eslint-disable no-undef */
import { Component, Vue, Ref } from 'vue-property-decorator';
import axios from 'axios';
import { Place } from '@/interfaces/place';
import WebSocketService from '@/services/websocketService';

@Component
export default class Map extends Vue {
  @Ref('map') readonly mapElement!: HTMLElement;
	map!: google.maps.Map;
	webSocket = WebSocketService.instance;

	markers: google.maps.Marker[] = [];
	infoWindows: google.maps.InfoWindow[] = []

	places: Place[] = [];

	created() {
		this.listenSockets();
	}

	async mounted() {
		await this.loadMarkers();
		this.loadMap();
	}

	listenSockets() {
		this.webSocket.listen('google-new-marker', (marker: Place) => {
			this.addMarker(marker);
		});
		this.webSocket.listen('google-delete-marker', (id: string) => {
			const marker = this.markers.find(marker => marker.getTitle() === id);
			if (marker) {
				marker.setMap(null);
			}
		});
		this.webSocket.listen('google-move-marker', (place: Place) => {
			const moveMarker = this.markers.find(marker => marker.getTitle() === place.id);
			if (moveMarker) {
				const latLng = new google.maps.LatLng(place.lat, place.lng);
				moveMarker.setPosition(latLng);
			}
		});
	}

	async loadMarkers() {
		const { data } = await axios.get<Place[]>(`${process.env.VUE_APP_API}/googlemaps`);
		this.places = data;
	}

	loadMap() {
		const latLng = new google.maps.LatLng(37.784679, -122.395936);
		const mapOptions: google.maps.MapOptions = {
			center: latLng,
			zoom: 13,
			mapTypeId: google.maps.MapTypeId.ROADMAP
		};
		this.map = new google.maps.Map(this.mapElement, mapOptions);

		this.map.addListener('click', (coors: any) => {
			const place: Place = {
				id: new Date().toISOString(),
				name: 'new place',
				lat: coors.latLng.lat(),
				lng: coors.latLng.lng(),
			}
			this.addMarker(place);
			this.webSocket.emit('google-new-marker', place);
		});

		for (const place of this.places) {
			this.addMarker(place);
		}
	}

	addMarker(place: Place) {
		const latLng = new google.maps.LatLng(place.lat, place.lng);
		const marker = new google.maps.Marker({
			map: this.map,
			animation: google.maps.Animation.DROP,
			position: latLng,
			draggable: true,
			title: place.id
		});
		this.markers.push(marker);

		const content = `<b>${place.name}</b>`;
		const infoWindow = new google.maps.InfoWindow({
			content
		});

		this.infoWindows.push(infoWindow);

		google.maps.event.addDomListener(marker, 'click', () => {
			this.infoWindows.forEach(infoW => infoW.close());
			infoWindow.open(this.map, marker);
		});

		google.maps.event.addDomListener(marker, 'dblclick', () => {
			marker.setMap(null);
			this.webSocket.emit('google-delete-marker', place.id);
		});

		google.maps.event.addDomListener(marker, 'drag', (coors: any) => {
			const newMarker = {
				id: marker.getTitle(),
				lat: coors.latLng.lat(),
				lng: coors.latLng.lng(),
				name: place.name
			}
			this.webSocket.emit('google-move-marker', newMarker);
		});
	}
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
#map {
	height: 500px;
	width: 100%;
}
</style>
