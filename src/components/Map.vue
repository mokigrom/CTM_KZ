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
    import { fromLonLat } from 'ol/proj';
    import VectorLayer from 'ol/layer/Vector';
    import VectorSource from 'ol/source/Vector';
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

    // Import Images
    

    const env = import.meta.env;
    
    type MenuKey = "railway" | "airport" | null;
    let menuLocationType = ref<MenuKey>(null);

    const railwayVisibility = ref<boolean>(true);
    const airportVisibility = ref<boolean>(true);

    interface Coordinate { lng: number, lat: number }
    interface Location { id: string, name: string, coordinates: Coordinate };
    interface Crime {
        id: number,
        locationId: string,
        coordinates: Coordinate,
        address: string,
        article: string,
        articleName: string,
        count: number,
        iconId: number
    };

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
        if (event.ctrlKey || event.metaKey) event.preventDefault();
    }, { passive: false });

    window.addEventListener('keydown', (event) => {
        if ((event.ctrlKey || event.metaKey) && ['+', '-', '=', '0'].includes(event.key)) {
            event.preventDefault();
        }
    });

    window.addEventListener('click', (event) => {
        event.preventDefault();
    });
    //

    function addLayer(source: object, style: Style, zIndex: number, renderBuffer: number = 100): VectorLayer {
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
        const view: View = map!.getView();
        view.setCenter(fromLonLat([lng, lat]));
        view.setZoom(zoom);
    }

    type LabelOffset = { x: number, y: number }

    function createMarkers(list: Location[], iconSrc: string, locationType: 'point' | 'label', iconScale: number = 0.06, labelOffset: LabelOffset = { x: 0, y: 28 }, labelFontSize: number = 14): Feature[] {
        return list.map(point => {
            const feature = new Feature({
                geometry: new Point(
                    fromLonLat([
                        point.coordinates.lng,
                        point.coordinates.lat
                    ])
                )
            });

            let style: Style;

            switch (locationType) {
                case 'point':
                    style = new Style({
                        image: new Icon({ src: iconSrc, scale: iconScale, anchor: [0.5, 0.5] }),
                    });
                    break;
                case 'label':
                    style = new Style({
                        text: new Text({
                            text: point.name,
                            offsetX: labelOffset.x,
                            offsetY: labelOffset.y,
                            font: `${labelFontSize}px sans-serif`,
                            fill: new Fill({ color: '#3a4868' }),
                            stroke: new Stroke({ color: 'white', width: 6 }),
                        })
                    });
                    break;
            }

            feature.set("id", point.id);
            feature.set("name", point.name);
            feature.set("coordinates", point.coordinates)
            feature.setStyle(style);

            return feature;
        });
    }

    function createCrimeMarkers(list: Crime[], iconSrc: string, pointType: 'point' | 'label', iconScale: number = 0.075, labelOffset: LabelOffset = { x: 18, y: 18 }, labelFontSize: number = 12, labelText: string | null = null): Feature[] {
        return list.map(point => {
            const feature = new Feature({
                geometry: new Point(
                    fromLonLat([
                        point.coordinates.lng,
                        point.coordinates.lat
                    ])
                )
            });

            let style: Style;

            switch (pointType) {
                case 'point':
                    style = new Style({
                        image: new Icon({ src: iconSrc, scale: iconScale, anchor: [0.5, 0.5] }),
                    });
                    break;
                case 'label':
                    style = new Style({
                        text: new Text({
                            text: labelText || String(point.count),
                            offsetX: labelOffset.x,
                            offsetY: labelOffset.y,
                            font: `${labelFontSize}px sans-serif`,
                            fill: new Fill({ color: '#3a4868' }),
                            stroke: new Stroke({ color: 'white', width: 6 }),
                        })
                    });
                    break;
            }

            feature.set("id", point.id);
            feature.set("name", point.article);
            feature.set("location_id", point.locationId);
            feature.set("address", point.address);
            feature.set("article", point.article);
            feature.set("article_name", point.articleName);
            feature.set("coordinates", point.coordinates);
            feature.set("count", point.count);
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

    async function getData(): Promise<Array<any>> {
        const response = await fetch(env.VITE_BACKEND_DATA_URL);

        const data = await response.json();
        return data;
    }

    onMounted( async () => {
        const alldata = await getData();
        const weaponData = alldata.filter(point => point.icon_id === 0);
        const drugData = alldata.filter(point => point.icon_id === 1);
        const fistData = alldata.filter(point => point.icon_id === 3);
        const fraudData = alldata.filter(point => point.icon_id === 4);
        const theftData = alldata.filter(point => point.icon_id === 5);
        const unknownData = alldata.filter(point => point.icon_id === 6);

        const railwayData = alldata.filter(point => point.location_id.includes("railway"));
        const airportData = alldata.filter(point => point.location_id.includes("airport"));
        console.log(railwayData);
        console.log(airportData);

        const weaponPointMarkers = new VectorLayer({
            source: new VectorSource({ features: createCrimeMarkers(weaponData, "/markers/weapon_marker.svg", 'point') }),
            zIndex: 7,
            minZoom: 14
        });

        const weaponLabelMarkers = new VectorLayer({
            source: new VectorSource({
                features: createCrimeMarkers(
                    weaponData, "/markers/weapon_marker.svg", 'label'
                )
            }),
            zIndex: 7,
            minZoom: 16
        })

        const drugPointMarkers = new VectorLayer({
            source: new VectorSource({ features: createCrimeMarkers(drugData, "/markers/drugs_marker.svg", 'point') }),
            zIndex: 7,
            minZoom: 14
        });

        const drugLabelMarkers = new VectorLayer({
            source: new VectorSource({
                features: createCrimeMarkers(
                    drugData, "/markers/drugs_marker.svg", 'label'
                )
            }),
            zIndex: 7,
            minZoom: 16
        });

        const fistPointMarkers = new VectorLayer({
            source: new VectorSource({ features: createCrimeMarkers(fistData, "/markers/fist_marker.svg", 'point') }),
            zIndex: 7,
            minZoom: 14
        });

        const fistLabelMarkers = new VectorLayer({
            source: new VectorSource({
                features: createCrimeMarkers(
                    fistData, "/markers/fist_marker.svg", 'label'
                )
            }),
            zIndex: 7,
            minZoom: 16
        });

        const fraudPointMarkers = new VectorLayer({
            source: new VectorSource({ features: createCrimeMarkers(fraudData, "/markers/fraud_marker.svg", 'point') }),
            zIndex: 7,
            minZoom: 14
        });

        const fraudLabelMarkers = new VectorLayer({
            source: new VectorSource({
                features: createCrimeMarkers(
                    fraudData, "/markers/fraud_marker.svg", 'label'
                )
            }),
            zIndex: 7,
            minZoom: 16
        });

        const theftPointMarkers = new VectorLayer({
            source: new VectorSource({ features: createCrimeMarkers(theftData, "/markers/theft_marker.svg", 'point') }),
            zIndex: 7,
            minZoom: 14
        });

        const theftLabelMarkers = new VectorLayer({
            source: new VectorSource({
                features: createCrimeMarkers(
                    theftData, "/markers/theft_marker.svg", 'label'
                )
            }),
            zIndex: 7,
            minZoom: 16
        });

        const unknownPointMarkers = new VectorLayer({
            source: new VectorSource({ features: createCrimeMarkers(unknownData, "/markers/unknown_marker.svg", 'point') }),
            zIndex: 7,
            minZoom: 14
        });

        const unknownLabelMarkers = new VectorLayer({
            source: new VectorSource({
                features: createCrimeMarkers(
                    unknownData, "/markers/unknown_marker.svg", 'label'
                )
            }),
            zIndex: 7,
            minZoom: 16
        });

        map = new Map({
            target: "js-map",
            layers: [
                OpenStreetMap,
                regionLayer,
                lakeLayer,
                borderLayer,
                maskLayer,
                railwayMarkers,
                airportMarkers,
                railwayLabels,
                airportLabels,
                unknownPointMarkers,
                unknownLabelMarkers,
                fistPointMarkers,
                fistLabelMarkers,
                fraudPointMarkers,
                theftPointMarkers,
                theftLabelMarkers,
                fraudLabelMarkers,
                drugPointMarkers,
                drugLabelMarkers,
                weaponPointMarkers,
                weaponLabelMarkers
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