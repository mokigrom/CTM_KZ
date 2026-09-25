<template>
    <div ref="mapRef" id="js-map" class="map"></div>
    <div id="location-options">
        <button @click="changeVisibility('railway')" :class="{ active: railwayVisibility }">
            <img src="/icons/railway_black.webp" v-if="!railwayVisibility">
            <img src="/icons/railway_white.webp" v-if="railwayVisibility">   
        </button>
        <button @click="changeVisibility('airport')" :class="{ active: airportVisibility }">
            <img src="/icons/airport_black.webp" v-if="!airportVisibility">
            <img src="/icons/airport_white.webp" v-if="airportVisibility">   
        </button>
    </div>
    <div id="menu-locations">
        <div id="types">
            <button id="railway" title="Железнодорожные вокзалы" @click="openLocationList('railway')" :class="{ active: menuLocationType === 'railway' }" v-if="railwayVisibility">
                <div class="icon">
                    <img src="/icons/railway_black.webp" alt="Railway Station Black Icon" v-if="menuLocationType !== 'railway'">
                    <img src="/icons/railway_white.webp" alt="Railway Station White Icon" v-if="menuLocationType === 'railway'">
                    <b>ЖД вокзалы</b>
                </div>
            </button>
            <button id="airport" title="Аэропорты" @click="openLocationList('airport')" :class="{ active: menuLocationType === 'airport'}" v-if="airportVisibility">
                <div class="icon">
                    <img src="/icons/airport_black.webp" alt="Airport Black Icon" v-if="menuLocationType !== 'airport'">
                    <img src="/icons/airport_white.webp" alt="Airport White Icon" v-if="menuLocationType === 'airport'">
                    <b>Аэропорты</b>
                </div>
            </button>
        </div>
        <div id="location-list" class="popup" v-if="menuLocationType">
            <ul v-if="menuLocationType === 'railway'">
                <li v-for="railway in railwayList" :key="railway.id">
                    <button :id="railway.id" @click="goToLocation(railway.coordinates.lng, railway.coordinates.lat)">{{ railway.name }}</button>
                </li>
            </ul>
            <ul v-else-if="menuLocationType === 'airport'">
                <li v-for="airport in airportList" :key="airport.id">
                    <button :id="airport.id" @click="goToLocation(airport.coordinates.lng, airport.coordinates.lat)">{{ airport.name }}</button>
                </li>
            </ul>
        </div>
    </div>
</template>

<style scoped>
    button {
        cursor: pointer;
        color: black;
        background-color: white;
        border: 1px solid black;
    }

    button.active {
        color: white;
        background-color: black;
    }

    #types {
        display: flex;
        flex-direction: row;
        justify-content: space-between;
    }

    #types button {
        display: flex;
        flex-direction: row;
        justify-content: space-evenly;
        appearance: none;
        width: 49%;

        padding-block: 0; padding-inline: 0;
    }

    #menu-locations, #location-options {
        display: flex;
        position: fixed;
        flex-direction: column;
        width: 360px;
        position: fixed;
        top: 8px;
        left: 8px;
        gap: 8px;
        z-index: 1;
    }

    #location-options {
        flex-direction: row;
        top: auto;
        bottom: 8px;
        width: initial;
    }

    #location-options button {
        padding: 4px;
    }

    #location-options img {
        display: block;
        width: 25px;
        height: 25px;
    }

    .map {
        position: fixed;
        top: 0;
        bottom: 0;
        width: 100%;
    }

    .icon {
        display: flex;
    }

    .icon img {
        display: block;
        width: 1.875rem;
        height: 1.875rem;
        margin-right: 8px;
        pointer-events: none;
        user-select: none;
    }

    .icon b {
        font-size: 0.75rem;
        margin-block: 0;
        margin: auto;
    }

    #location-list {
        display: flex;
        flex-direction: column;
        padding: 7px;
        width: calc(100% - 16px);
        overflow-y: auto;
        max-height: 80vh;
    }

    #location-list::-webkit-scrollbar { appearance: none; width: 3px; scrollbar-width: none; }
    #location-list::-webkit-scrollbar-track { background-color: transparent; }
    #location-list::-webkit-scrollbar-thumb { background-color: black; }

    #location-list ul {
        display: flex;
        flex-direction: column;
        list-style-type: none;
        margin: 0;
        padding: 0;
        gap: 0.25rem;
    }

    #location-list button {
        display: block;
        text-align: left;
        padding: 0.25rem 0.5rem;
        width: calc(100% - 0.125rem);
        border: none;
    }

    #location-list button:hover { text-decoration: underline; }

    .popup {
        background-color: white;
        border: 1px solid black;
    }
</style>

<script setup lang="ts">
    import { ref, onMounted, onUnmounted } from 'vue';
    import Map from 'ol/Map';
    import OSM from 'ol/source/OSM';
    import { Point } from 'ol/geom';
    import TileLayer from 'ol/layer/Tile';
    import View from 'ol/View';
    import { defaults as defaultInteractions } from 'ol/interaction/defaults';
    import { fromLonLat } from 'ol/proj';
    import ImageLayer from 'ol/layer/Image';
    import VectorLayer from 'ol/layer/Vector';
    import VectorSource from 'ol/source/Vector';
    import Static from 'ol/source/ImageStatic';
    import GeoJSON from 'ol/format/GeoJSON'
    import { Style, Icon, Text } from 'ol/style';
    import Fill from 'ol/style/Fill';
    import Stroke from 'ol/style/Stroke';

    import { Feature } from 'ol';

    // Import GeoJSON
    import lakes from '../assets/geo/lakes.json'
    import kzRegions from '../assets/geo/regions.json';
    import kzBorder from '../assets/geo/border.json';
    import mask from '../assets/geo/mask.json';

    // Import Locations
    import railwayResponse from '../assets/locations/railways.json';
    import airportResponse from '../assets/locations/airports.json';

    const env = import.meta.env;

    const mapRef: Object = ref<Map | null>(null);
    
    type MenuKey = "railway" | "airport" | null;
    let menuLocationType = ref<MenuKey>(null);

    const railwayVisibility = ref<boolean>(true);
    const airportVisibility = ref<boolean>(true);

    interface Coordinate { lng: number, lat: number }
    interface Location { id: "string", name: string, coordinates: Coordinate[] };
    const railwayList = ref<Location[]>(railwayResponse);
    const airportList = ref<Location[]>(airportResponse);

    let map: Map | null = null;

    const OpenStreetMap: TileLayer = new TileLayer({
        source: new OSM({
            url: env.VITE_MAP_URL
        })
    });

    const regionLayer = addLayer(
        kzRegions,
        new Style({ stroke: new Stroke({ color: "#596d9f", width: 1.5 }) }),
        3
    );

    const maskLayer = addLayer(
        mask,
        new Style({ fill: new Fill({ color: "#ffffff" }) }),
        2,
        4960
    );

    const lakeLayer = addLayer(
        lakes,
        new Style({ fill: new Fill({ color: "#addeff" }) }),
        3
    );

    const borderLayer = addLayer(
        kzBorder,
        new Style({ stroke: new Stroke({ color: "#596d9f", width: 3 }) }),
        3,
        4960
    );

    const railwayMarkers = new VectorLayer({
        source: new VectorSource({ features: createMarkers(railwayList.value, "/markers/railway_marker.svg", 'point') }),
        zIndex: 4
    });

    const airportMarkers = new VectorLayer({
        source: new VectorSource({ features: createMarkers(airportList.value, "/markers/airport_marker.svg", 'point') }),
        zIndex: 5
    });

    const railwayLabels = new VectorLayer({
        source: new VectorSource({ features: createMarkers(railwayList.value, "/markers/railway_marker.svg", 'label') }),
        zIndex: 6,
        minZoom: 10
    });

    const airportLabels = new VectorLayer({
        source: new VectorSource({ features: createMarkers(airportList.value, "/markers/airport_marker.svg", 'label') }),
        zIndex: 6,
        minZoom: 10
    });

    // Disable Page Scale
    window.addEventListener('wheel', (event) => {
        if (event.ctrlKey || event.metaKey) event.preventDefault()
    }, { passive: false });

    window.addEventListener('keydown', (event) => {
        if ((event.ctrlKey || event.metaKey) && ['+', '-', '=', '0'].includes(event.key)) {
            event.preventDefault()
        }
    });
    //

    function addLayer(source: VectorSource, style: Style, zIndex: number, renderBuffer: number = 100): VectorLayer {
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

    function openLocationList(key: MenuKey): void {
        if (menuLocationType.value === key) menuLocationType.value = null
        else menuLocationType.value = key;
    }

    function goToLocation(lng: number, lat: number, zoom: number = 16.5) {
        const view: View = map.getView();
        // view.animate({
        //     center: fromLonLat([lng, lat]),
        //     zoom: zoom,
        //     duration: 1000
        // });
        view.setCenter(fromLonLat([lng, lat]));
        view.setZoom(zoom);
    }

    function createMarkers(list: Location[], iconSrc: string, locationType: 'point' | 'label'): Feature[] {
        return list.map(location => {
            const feature = new Feature({
                geometry: new Point(
                    fromLonLat([
                        location.coordinates.lng,
                        location.coordinates.lat
                    ])
                )
            })

            let style: Style;

            switch (locationType) {
                case 'point':
                    style = new Style({
                        image: new Icon({ src: iconSrc, scale: 0.05, anchor: [0.5, 1] }),
                    });
                    break;
                case 'label':
                    style = new Style({
                        text: new Text({
                            text: location.name,
                            offsetY: 10,
                            font: '14px sans-serif',
                            fill: new Fill({ color: '#3a4868' }),
                            stroke: new Stroke({ color: 'white', width: 6 }),
                       })
                    });
                    break;
            }

            feature.set("id", location.id);
            feature.set("name", location.name);
            feature.setStyle(style);

            return feature;
        });
    }

    function changeVisibility(type: 'railway' | 'airport'): void {
        switch (type) {
            case 'railway':
                railwayVisibility.value = !railwayVisibility.value;
                railwayMarkers.setVisible(railwayVisibility.value)
                railwayLabels.setVisible(railwayVisibility.value)
                if (menuLocationType.value === 'railway') menuLocationType.value = null;
                break;
            case 'airport':
                airportVisibility.value = !airportVisibility.value;
                airportMarkers.setVisible(airportVisibility.value)
                airportLabels.setVisible(airportVisibility.value)
                if (menuLocationType.value === 'airport') menuLocationType.value = null;
                break;
        }
    }

    onMounted( () => {
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
                maskLayer,
                railwayMarkers,
                airportMarkers,
                railwayLabels,
                airportLabels
            ],
            view: new View({
                center: fromLonLat([66.3, 48.5]),
                zoom: 5,
                maxZoom: 18,
                extent: [
                    ...fromLonLat([42.2, 34.0]),
                    ...fromLonLat([92.5, 58.5])
                ]
            })
        });
    });

    onUnmounted(() => map?.setTarget(undefined));
</script>