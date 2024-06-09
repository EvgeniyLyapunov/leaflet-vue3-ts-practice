<template>
	<div class="map-wrapper">
		<div id="map"></div>
	</div>
</template>

<script setup lang="ts">
	import * as L from 'leaflet';
	import 'leaflet/dist/leaflet.css';
	import 'leaflet.markercluster/dist/MarkerCluster.css';
	import 'leaflet.markercluster/dist/MarkerCluster.Default.css';
	import 'leaflet.markercluster';
	import { onMounted, ref } from 'vue';

	interface IMarker {
		name: string;
		lat: number;
		lon: number;
	}

	const listOfMarkers = ref<IMarker[]>([
		{ name: 'first point', lat: 45.098951, lon: 39.018851 },
		{ name: 'second point', lat: 45.30372, lon: 40.94905 },
		{ name: 'third point', lat: 43.60957, lon: 39.72901 },
		{ name: 'forth point', lat: 46.631531, lon: 38.650011 },
		{ name: 'fifth point', lat: 45.118367, lon: 37.426781 },
	]);

	const markerList: L.Marker[] = listOfMarkers.value.map((item) => {
		return L.marker([item.lat, item.lon], { title: item.name }).bindPopup(item.name);
	});

	const allCameras_MarkersListLayer = L.layerGroup([...markerList]);
	let selectedCameras_MarkersListLayer = L.layerGroup([]);

	const baseLayer = L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
		maxZoom: 19,
	});

	const baseMaps = {
		OpenStreetMap: baseLayer,
	};

	const overlayMaps = {
		'Все камеры': allCameras_MarkersListLayer,
	};

	const defaultIcon = L.icon({
		iconUrl:
			'https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-blue.png',
		shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png',
		iconSize: [25, 41],
		iconAnchor: [12, 41],
		popupAnchor: [1, -34],
		shadowSize: [41, 41],
	});

	const selectedIcon = L.icon({
		iconUrl:
			'https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-red.png',
		shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/0.7.7/images/marker-shadow.png',
		iconSize: [25, 41],
		iconAnchor: [12, 41],
		popupAnchor: [1, -34],
		shadowSize: [41, 41],
	});

	let selectedMarkers = new Set<L.Marker<any>>();

	onMounted(() => {
		const map = L.map('map', {
			center: [45.098951, 39.018851],
			zoom: 7,
			layers: [baseLayer, allCameras_MarkersListLayer],
		});

		const layerControl = L.control.layers(baseMaps, overlayMaps).addTo(map);

		const markers = L.markerClusterGroup(); // Создаем группу маркеров

		markerList.forEach((m) => {
			m.setIcon(defaultIcon); // Устанавливаем начальную иконку
			m.on('click', () => {
				// Удаляем слой выбранных маркеров из контрола слоев
				layerControl.removeLayer(selectedCameras_MarkersListLayer);

				if (m.getIcon() === selectedIcon) {
					m.setIcon(defaultIcon); // Сброс на начальную иконку, если маркер уже выбран
					selectedMarkers.delete(m);
				} else {
					m.setIcon(selectedIcon); // Установка выбранной иконки, если маркер не выбран
					selectedMarkers.add(m);
				}

				// Обновляем слой выбранных маркеров
				selectedCameras_MarkersListLayer.clearLayers();
				selectedMarkers.forEach((marker) => {
					selectedCameras_MarkersListLayer.addLayer(marker);
				});

				// Добавляем слой выбранных маркеров обратно в контрол слоев
				layerControl.addOverlay(selectedCameras_MarkersListLayer, 'Выбранные камеры');
			});
			m.addTo(markers); // Добавляем маркеры в группу
		});

		map.addLayer(markers); // Добавляем маркеры в группу

		// Добавляем возможность выделения области
		let selectionRectangle: L.Rectangle | null = null;
		let startLatLng: L.LatLng | null = null;

		map.on('mousedown', (e: L.LeafletMouseEvent) => {
			// Проверяем, нажата ли клавиша Shift
			if (e.originalEvent.shiftKey) {
				// Отключаем зум карты
				map.dragging.disable();
				map.boxZoom.disable();
				map.scrollWheelZoom.disable();
				map.keyboard.disable();
				if (map.tap) map.tap.disable();

				startLatLng = e.latlng;
				selectionRectangle = L.rectangle(L.latLngBounds(startLatLng, startLatLng), {
					color: '#007bff',
					weight: 2,
					fillOpacity: 0.2,
				}).addTo(map);
			}
		});

		map.on('mousemove', (e: L.LeafletMouseEvent) => {
			if (selectionRectangle && startLatLng) {
				selectionRectangle.setBounds(L.latLngBounds(startLatLng, e.latlng));
			}
		});

		map.on('mouseup', () => {
			if (selectionRectangle) {
				// Включаем зум карты
				map.dragging.enable();
				map.boxZoom.enable();
				map.scrollWheelZoom.enable();
				map.keyboard.enable();
				if (map.tap) map.tap.enable();

				// Получаем границы выделенной области
				const bounds = selectionRectangle.getBounds();

				// Находим маркеры внутри выделенной области
				const selectedMarkersByRectangle = markers
					.getLayers()
					.filter((layer): layer is L.Marker => {
						return (
							(layer as L.Marker).getLatLng() && bounds.contains((layer as L.Marker).getLatLng())
						);
					});

				// Изменяем иконку выбранных маркеров
				selectedMarkersByRectangle.forEach((marker) => {
					marker.setIcon(selectedIcon);
					selectedMarkers.add(marker);
				});

				// Выводим информацию о выбранных маркерах (например, в консоль)
				console.log(
					'Выбраны маркеры в области выделения:',
					selectedMarkersByRectangle.map((marker) => marker.options.title)
				);

				console.log(
					'Выбраны маркеры:',
					Array.from(selectedMarkers).map((marker) => marker.options.title)
				);

				// Удаляем прямоугольник выделения
				selectionRectangle.remove();
				selectionRectangle = null;

				// Обновляем слой выбранных маркеров
				selectedCameras_MarkersListLayer.clearLayers();
				selectedMarkers.forEach((marker) => {
					selectedCameras_MarkersListLayer.addLayer(marker);
				});

				// Обновляем контрол слоев
				layerControl.removeLayer(selectedCameras_MarkersListLayer);
				layerControl.addOverlay(selectedCameras_MarkersListLayer, 'Выбранные камеры');
			}
		});

		// скрываем копирайт
		const copyEl: HTMLElement | null = document.querySelector('.leaflet-bottom.leaflet-right');
		if (copyEl !== null) {
			copyEl.style.display = 'none';
		}
	});
</script>

<style scoped>
	.map-wrapper {
		width: 100%;
		height: 100vh;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	#map {
		width: 700px;
		height: 550px;
		border: 3px solid teal;
		border-radius: 4px;
	}
</style>
