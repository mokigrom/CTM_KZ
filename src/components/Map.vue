<template>
    <div ref="mapRef" id="js-map" class="map"></div>
</template>

<script setup lang="ts">
    import { ref, onMounted, onUnmounted } from 'vue';
    import Map from 'ol/Map';
    import OSM from 'ol/source/OSM';
    import TileLayer from 'ol/layer/Tile';
    import View from 'ol/View';
    import { defaults as defaultInteractions } from 'ol/interaction/defaults';
    import { fromLonLat } from 'ol/proj';
    import ImageLayer from 'ol/layer/Image';
    import VectorLayer from 'ol/layer/Vector';
    import VectorSource from 'ol/source/Vector';
    import Static from 'ol/source/ImageStatic';
    import GeoJSON from 'ol/format/GeoJSON'
    import Style from 'ol/style/Style';
    import Fill from 'ol/style/Fill';
    import Stroke from 'ol/style/Stroke';

    import lakes from '../assets/geo/lakes.json'
    import kzRegions from '../assets/geo/regions.json';
    import kzBorder from '../assets/geo/border.json';
    import mask from '../assets/geo/mask.json';

    const mapRef: Object = ref(null);
    let map: Map | null = null;

    const OpenStreetMap: TileLayer = new TileLayer({
        source: new OSM({
            url: "https://api.thunderforest.com/transport/{z}/{x}/{y}.png?apikey=5203085883c2470a9fa54bab98f3e0de"
        })
    });

    const regionLayer = getLayer(
        kzRegions,
        new Style({ stroke: new Stroke({ color: "#596d9f", width: 1.5 }) }),
        3
    );

    const maskLayer = getLayer(
        mask,
        new Style({ fill: new Fill({ color: "#ffffff" }) }),
        2,
        4960
    );

    const lakeLayer = getLayer(
        lakes,
        new Style({ fill: new Fill({ color: "#addeff" }) }),
        3
    );

    const borderLayer = getLayer(
        kzBorder,
        new Style({ stroke: new Stroke({ color: "#596d9f", width: 3 }) }),
        3
    );

    // Disable Scale
    window.addEventListener('wheel', (event) => {
        if (event.ctrlKey || event.metaKey) event.preventDefault()
    }, { passive: false });

    window.addEventListener('keydown', (event) => {
        if ((event.ctrlKey || event.metaKey) && ['+', '-', '=', '0'].includes(event.key)) {
            event.preventDefault()
        }
    });
    //

    function getLayer(source: VectorSource, style: Style, zIndex: number, renderBuffer: number = 100): VectorLayer {
        return new VectorLayer({
            source: new VectorSource({
            features: new GeoJSON().readFeatures(source, {
                dataProjection: 'EPSG:4326',
                featureProjection: 'EPSG:3857',
            }),
        }),
        style: style,
        zIndex: zIndex,
        renderBuffer: renderBuffer
    });
    }

    function goToLocation(map: Map, lng: number, lat: number, zoom: number = 15) {
        const view: View = map.getView();
    }

    onMounted(() => {
        map = new Map({
            target: mapRef.value!,
            // interactions: defaultInteractions({
            //     mouseWheelZoom: false,
            //     doubleClickZoom: false,
            //     pinchZoom: false,
            //     keyboard: false
            // }),
            layers: [
                OpenStreetMap,
                regionLayer,
                lakeLayer,
                borderLayer,
                maskLayer
            ],
            view: new View({
                center: fromLonLat([66.3, 48.5]),
                maxZoom: 18,
                zoom: 1,
                extent: [
                    ...fromLonLat([42.8, 34.0]),
                    ...fromLonLat([93.0, 58.5])
                ]
            })
        });
    });

    onUnmounted(() => map?.setTarget(undefined));
</script>

<style scoped>
    .map {
        position: fixed;
        top: 0;
        bottom: 0;
        width: 100%;
    }
</style>