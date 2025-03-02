<template>
	<div
		id="map-container"
		ref="container"
		:class="{ select: selectMode, hover: hoveredFeature || hoveredCluster }"
	></div>
</template>

<script lang="ts">
import 'maplibre-gl/dist/maplibre-gl.css';
import maplibre, {
	MapboxGeoJSONFeature,
	MapLayerMouseEvent,
	AttributionControl,
	NavigationControl,
	GeolocateControl,
	LngLatBoundsLike,
	GeoJSONSource,
	CameraOptions,
	LngLatLike,
	AnyLayer,
	Map,
} from 'maplibre-gl';
import MapboxGeocoder from '@mapbox/mapbox-gl-geocoder';
import '@mapbox/mapbox-gl-geocoder/dist/mapbox-gl-geocoder.css';
import { ref, watch, PropType, onMounted, onUnmounted, defineComponent, toRefs, computed, WatchStopHandle } from 'vue';
import { useI18n } from 'vue-i18n';

import { ShowSelect } from '@directus/shared/types';
import getSetting from '@/utils/get-setting';
import { useAppStore } from '@/stores';
import { BoxSelectControl, ButtonControl } from '@/utils/geometry/controls';
import { getBasemapSources, getStyleFromBasemapSource } from '@/utils/geometry/basemap';

export default defineComponent({
	components: {},
	props: {
		data: {
			type: Object as PropType<GeoJSON.FeatureCollection>,
			required: true,
		},
		source: {
			type: Object as PropType<GeoJSONSource>,
			required: true,
		},
		layers: {
			type: Array as PropType<AnyLayer[]>,
			default: () => [],
		},
		camera: {
			type: Object as PropType<CameraOptions & { bbox: any }>,
			default: () => ({} as any),
		},
		bounds: {
			type: Array as unknown as PropType<GeoJSON.BBox>,
			default: undefined,
		},
		featureId: {
			type: String,
			default: undefined,
		},
		selection: {
			type: Array as PropType<Array<string | number>>,
			default: () => [],
		},
		showSelect: {
			type: String as PropType<ShowSelect>,
			default: 'multiple',
		},
	},
	emits: ['moveend', 'featureclick', 'featureselect', 'fitdata', 'updateitempopup'],
	setup(props, { emit }) {
		const { t } = useI18n();
		const appStore = useAppStore();
		let map: Map;
		const hoveredFeature = ref<MapboxGeoJSONFeature>();
		const hoveredCluster = ref<boolean>();
		const selectMode = ref<boolean>();
		const container = ref<HTMLElement>();
		const unwatchers = [] as WatchStopHandle[];
		const { sidebarOpen, basemap } = toRefs(appStore);
		const mapboxKey = getSetting('mapbox_key');
		const basemaps = getBasemapSources();
		const style = computed(() => {
			const source = basemaps.find((source) => source.name === basemap.value) ?? basemaps[0];
			return getStyleFromBasemapSource(source);
		});

		const attributionControl = new AttributionControl({ compact: true });
		const navigationControl = new NavigationControl({
			showCompass: false,
		});
		const geolocateControl = new GeolocateControl();
		const fitDataControl = new ButtonControl('mapboxgl-ctrl-fitdata', () => {
			emit('fitdata');
		});
		const boxSelectControl = new BoxSelectControl({
			boxElementClass: 'map-selection-box',
			selectButtonClass: 'mapboxgl-ctrl-select',
			layers: ['__directus_polygons', '__directus_points', '__directus_lines'],
		});
		let geocoderControl: MapboxGeocoder | undefined;
		if (mapboxKey) {
			const marker = document.createElement('div');
			marker.className = 'mapboxgl-user-location-dot mapboxgl-search-location-dot';
			geocoderControl = new MapboxGeocoder({
				accessToken: mapboxKey,
				collapsed: true,
				marker: { element: marker } as any,
				flyTo: { speed: 1.4 },
				mapboxgl: maplibre as any,
				placeholder: t('layouts.map.find_location'),
			});
		}
		onMounted(() => {
			setupMap();
		});
		onUnmounted(() => {
			map.remove();
		});

		return { container, hoveredFeature, hoveredCluster, selectMode };

		function setupMap() {
			map = new Map({
				container: 'map-container',
				style: style.value,
				attributionControl: false,
				dragRotate: false,
				...props.camera,
				...(mapboxKey ? { accessToken: mapboxKey } : {}),
			});

			if (geocoderControl) {
				map.addControl(geocoderControl as any, 'top-right');
			}
			map.addControl(attributionControl, 'top-right');
			map.addControl(navigationControl, 'top-left');
			map.addControl(geolocateControl, 'top-left');
			map.addControl(fitDataControl, 'top-left');
			map.addControl(boxSelectControl, 'top-left');

			map.on('load', () => {
				watch(() => style.value, updateStyle);
				watch(() => props.bounds, fitBounds);
				const activeLayers = ['__directus_polygons', '__directus_points', '__directus_lines'];
				for (const layer of activeLayers) {
					map.on('click', layer, onFeatureClick);
					map.on('mousemove', layer, updatePopup);
					map.on('mouseleave', layer, updatePopup);
				}
				map.on('move', updatePopupLocation);
				map.on('click', '__directus_clusters', expandCluster);
				map.on('mousemove', '__directus_clusters', hoverCluster);
				map.on('mouseleave', '__directus_clusters', hoverCluster);
				map.on('select.enable', () => (selectMode.value = true));
				map.on('select.disable', () => (selectMode.value = false));
				map.on('select.end', (event: MapLayerMouseEvent) => {
					const ids = event.features?.map((f) => f.id);
					emit('featureselect', { ids, replace: !event.alt });
				});
				map.on('moveend', (event) => {
					if (!event.originalEvent) {
						return;
					}
					emit('moveend', {
						center: map.getCenter(),
						zoom: map.getZoom(),
						bearing: map.getBearing(),
						pitch: map.getPitch(),
						bbox: map.getBounds().toArray().flat(),
					});
				});
				startWatchers();
			});

			watch(
				() => sidebarOpen.value,
				(opened) => {
					if (!opened) setTimeout(() => map.resize(), 300);
				}
			);
			setTimeout(() => map.resize(), 300);
		}

		function fitBounds() {
			const bbox = props.data.bbox;
			if (map && bbox) {
				map.fitBounds(bbox as LngLatBoundsLike, {
					padding: 100,
					speed: 1.3,
					maxZoom: 14,
				});
			}
		}

		function updateStyle(style: any) {
			unwatchers.forEach((unwatch) => unwatch());
			unwatchers.length = 0;
			map.setStyle(style, { diff: false });
			map.once('styledata', startWatchers);
		}

		function startWatchers() {
			unwatchers.push(
				watch(() => props.source, updateSource, { immediate: true }),
				watch(() => props.selection, updateSelection, { immediate: true }),
				watch(() => props.layers, updateLayers),
				watch(() => props.data, updateData)
			);
		}

		function updateData(newData: any) {
			const source = map.getSource('__directus');
			(source as GeoJSONSource).setData(newData);
			updateSelection(props.selection, undefined);
		}

		function updateSource(newSource: GeoJSONSource) {
			const layersId = new Set(map.getStyle().layers?.map(({ id }) => id));
			for (const layer of props.layers) {
				if (layersId.has(layer.id)) {
					map.removeLayer(layer.id);
				}
			}
			if (props.featureId) {
				(newSource as any).promoteId = props.featureId;
			} else {
				(newSource as any).generateId = true;
			}
			if (map.getStyle().sources?.['__directus']) {
				map.removeSource('__directus');
			}
			map.addSource('__directus', { ...newSource, data: props.data });
			map.once('sourcedata', () => {
				setTimeout(() => props.layers.forEach((layer) => map.addLayer(layer)));
			});
		}

		function updateLayers(newLayers?: AnyLayer[], previousLayers?: AnyLayer[]) {
			const currentMapLayersId = new Set(map.getStyle().layers?.map(({ id }) => id));
			previousLayers?.forEach((layer) => {
				if (currentMapLayersId.has(layer.id)) map.removeLayer(layer.id);
			});
			newLayers?.forEach((layer) => {
				map.addLayer(layer);
			});
		}

		function updateSelection(newSelection?: (string | number)[], previousSelection?: (string | number)[]) {
			previousSelection?.forEach((id) => {
				map.setFeatureState({ id, source: '__directus' }, { selected: false });
				map.removeFeatureState({ id, source: '__directus' });
			});
			newSelection?.forEach((id) => {
				map.setFeatureState({ id, source: '__directus' }, { selected: true });
			});
		}

		function onFeatureClick(event: MapLayerMouseEvent) {
			const feature = event.features?.[0];
			const replace = props.showSelect === 'multiple' ? false : !event.originalEvent.altKey;
			if (feature && props.featureId) {
				if (boxSelectControl.active()) {
					emit('featureselect', { ids: [feature.id], replace });
				} else {
					emit('featureclick', { id: feature.id, replace });
				}
			}
		}

		function updatePopup(event: MapLayerMouseEvent) {
			const feature = map.queryRenderedFeatures(event.point, {
				layers: ['__directus_polygons', '__directus_points', '__directus_lines'],
			})[0];

			const previousId = hoveredFeature.value?.id;
			const featureChanged = previousId !== feature?.id;
			if (previousId && featureChanged) {
				map.setFeatureState({ id: previousId, source: '__directus' }, { hovered: false });
			}
			if (feature && feature.properties) {
				if (feature.geometry.type === 'Point') {
					const { x, y } = map.project(feature.geometry.coordinates as LngLatLike);
					const rect = map.getContainer().getBoundingClientRect();
					emit('updateitempopup', { position: { x: rect.x + x, y: rect.y + y } });
				} else {
					const { clientX: x, clientY: y } = event.originalEvent;
					emit('updateitempopup', { position: { x, y } });
				}
				if (featureChanged) {
					map.setFeatureState({ id: feature.id, source: '__directus' }, { hovered: true });
					hoveredFeature.value = feature;
					emit('updateitempopup', { item: feature.id });
				}
			} else {
				if (featureChanged) {
					hoveredFeature.value = feature;
					emit('updateitempopup', { item: null });
				}
			}
		}

		function updatePopupLocation(event: MapLayerMouseEvent) {
			if (hoveredFeature.value && event.originalEvent) {
				const { x, y } = event.originalEvent;
				emit('updateitempopup', { position: { x, y } });
			}
		}

		function expandCluster(event: MapLayerMouseEvent) {
			const features = map.queryRenderedFeatures(event.point, {
				layers: ['__directus_clusters'],
			});
			const clusterId = features[0]?.properties?.cluster_id;
			const source = map.getSource('__directus') as GeoJSONSource;
			source.getClusterExpansionZoom(clusterId, (err: any, zoom: number) => {
				if (err) return;
				map.flyTo({
					center: (features[0].geometry as GeoJSON.Point).coordinates as LngLatLike,
					zoom: zoom,
					speed: 1.3,
				});
			});
		}

		function hoverCluster(event: MapLayerMouseEvent) {
			if (event.type == 'mousemove') {
				hoveredCluster.value = true;
			} else {
				hoveredCluster.value = false;
			}
		}
	},
});
</script>

<style lang="scss" scoped>
#map-container.hover :deep(.mapboxgl-canvas-container) {
	cursor: pointer !important;
}

#map-container.select :deep(.mapboxgl-canvas-container) {
	cursor: crosshair !important;
}

#map-container {
	position: relative;
	width: 100%;
	height: 100%;
}
</style>
