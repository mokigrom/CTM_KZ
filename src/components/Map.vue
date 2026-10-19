<template>
    <div ref="mapRef" id="js-map" class="map"></div>
    <div id="location-options">
        <button @click="changeVisibility('railway'); closeCrimePanel();" :class="{ active: railwayVisibility }">
            <img src="/icons/railway_black.webp" v-if="!railwayVisibility">
            <img src="/icons/railway_white.webp" v-if="railwayVisibility">   
        </button>
        <button @click="changeVisibility('airport'); closeCrimePanel();" :class="{ active: airportVisibility }">
            <img src="/icons/airport_black.webp" v-if="!airportVisibility">
            <img src="/icons/airport_white.webp" v-if="airportVisibility">   
        </button>
    </div>
    <div id="menu-locations">
        <div id="types">
            <div style="display: flex; justify-content: space-between; flex-direction: row; width: 100%; gap: 4px;">
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
        </div>
        <aside
            v-if="selectedCrimeLocation"
            id="crime-panel"
            class="popup"
            aria-live="polite"
        >
            <div class="crime-panel-header">
                <div style="display: flex; flex-direction: row; gap: 8px;">
                    <img 
                        :src="locationMarkerByType[selectedCrimeLocationType]"
                        width="40"
                    >
                    <h2>{{ selectedCrimeLocation }}</h2>
                </div>
                <div style="display: flex; align-items: flex-start;">
                    <button
                        type="button"
                        class="close-button"
                        aria-label="Закрыть"
                        @click="closeCrimePanel"
                    >
                        <span>&times;</span>
                    </button>
                </div>
            </div>
            <div class="crime-table-container">
                <table>
                    <thead>
                        <tr>
                            <th style="font-size: 0.86rem; text-align: center;">№</th>
                            <th style="font-size: 0.86rem;">Наименование</th>
                            <th style="font-size: 0.8rem;">Статья</th>
                            <th style="font-size: 0.86rem; text-align: center;">Кол-во</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="crime, num in selectedCrimes" :key="crime.id">
                            <td style="padding: 0; font-size: 0.86rem; text-align: center; position: relative;">
                                <b style="position: absolute; top: 6px; left: 0; width: 100%;">{{ num + 1 }}</b>
                            </td>
                            <td style="font-size: 0.86rem;">
                                <div class="crime-name-cell">
                                    <span>{{ crime.article_name }}</span>
                                    <img :src="crimeMarkerByIconId[crime.icon_id] ?? crimeMarkerByIconId[6]">
                                </div>
                            </td>
                            <td style="font-size: 0.8rem;"><i>{{ crime.article }}</i></td>
                            <td style="font-size: 0.86rem; text-align: center;">{{ crime.count }}</td>
                        </tr>
                        <tr style="background-color: grey; color: white; border-color: black;">
                            <td style="border: none;"></td>
                            <td style="font-size: 0.96rem; text-align: left; border: none;"><b>Итого</b></td>
                            <td style="border: none;"></td>
                            <td style="font-size: 0.96rem; text-align: center; border: none;"><b>{{ selectedCrimes.reduce((sum, item) => sum + item.count, 0) }}</b></td>
                        </tr>
                        <tr v-if="selectedCrimes.length === 0">
                            <td colspan="3">Данные отсутствуют</td>
                        </tr>
                    </tbody>
                </table>
            </div>
        </aside>
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
        <div
            v-if="hoveredCrime || hoveredLocationMarker"
            id="crime-tooltip"
            :style="{ left: `${crimeTooltipPosition.x}px`, top: `${crimeTooltipPosition.y}px` }"
            aria-live="polite"
        >
            <b v-if="hoveredLocationMarker">{{ hoveredLocationMarker.name }}</b>
            <template v-else-if="hoveredCrime">
                <b>{{ hoveredCrime.article_name }}</b>
                <span><i>{{ hoveredCrime.article }}</i></span>
                <span>Количество: {{ hoveredCrime.count }}</span>
            </template>
        </div>
    </div>
    <div id="open-hovered-location-container">
        <button
        v-if="nearbyLocation && !selectedCrimeLocation"
        id="open-hovered-location-button"
        type="button"
        @click="openNearbyLocationCrimePanel"
    >
        <b>Открыть таблицу</b>
    </button>
    </div>
    <div id="watermark">
        <img src="/markers/airport_marker.svg">
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

    #open-hovered-location-container {
        position: fixed;
        top: 8px; right: 8px;
        z-index: 2;
    }

    #open-hovered-location-button {
        max-width: calc(100vw - 16px);
        padding: 6px 12px;
        text-align: left;
    }

    #crime-tooltip {
        position: fixed;
        z-index: 2;
        display: flex;
        flex-direction: column;
        gap: 4px;
        max-width: min(320px, calc(100vw - 24px));
        padding: 8px 10px;
        color: #222;
        background-color: white;
        border: 1px solid #596d9f;
        box-shadow: 0 2px 8px rgb(0 0 0 / 20%);
        font-size: 0.875rem;
        pointer-events: none;
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

    button b {
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

    #crime-panel {
        position: fixed;
        top: 8px;
        right: 8px;
        z-index: 1;
        display: flex;
        flex-direction: column;
        width: min(480px, calc(100vw - 32px));
        max-height: calc(100vh - 32px);
        padding: 12px;
        box-sizing: border-box;
    }

    .crime-panel-header {
        display: flex;
        align-items: center;
        justify-content: space-between;
        position: relative;
        margin-bottom: 12px;
        gap: 12px;
    }

    .crime-panel-header.collapsed {
        margin-bottom: 0;
    }

    .crime-panel-header h2 {
        margin: auto 12px;
        margin-left: 0;
        font-size: 1.125rem;
    }

    .location-type {
        display: flex;
        align-items: center;
        gap: 6px;
        margin-left: auto;
        font-size: 0.875rem;
    }

    .location-type img {
        width: 28px;
        height: 28px;
        object-fit: contain;
        user-select: none;
        pointer-events: none;
    }

    .close-button {
        flex: 0 0 auto;
        position: relative;
        justify-content: center;
        align-items: center;
        font-size: 1.25rem;
        padding: 0;
        line-height: 1;
        width: 24px; height: 24px;
        font-size: 24px;
        background-color: black;
        color: white;
        border-radius: 8px;
    }

    .close-button span {
        position: absolute;
        top: 50%; left: 50%;
        transform: translateX(-50%) translateY(-50%);
    }

    .collapse-button {
        flex: 0 0 auto;
        font-size: 1.125rem;
        line-height: 1;
        border: none;
    }

    .crime-table-container {
        overflow: auto;
    }

    #crime-panel table {
        width: 100%;
        border-collapse: collapse;
        text-align: left;
    }

    #crime-panel th,
    #crime-panel td {
        padding: 6px;
        border: 1px solid #d0d0d0;
    }

    .crime-name-cell {
        display: flex;
        align-items: flex-start;
        justify-content: space-between;
        gap: 8px;
    }

    .crime-name-cell span {
        min-width: 0;
        overflow-wrap: anywhere;
    }

    .crime-name-cell img {
        flex: 0 0 auto;
        width: 28px;
        height: 28px;
        object-fit: contain;
        user-select: none;
        pointer-events: none;
    }

    #crime-panel th:last-child,
    #crime-panel td:last-child {
        text-align: right;
    }

    #crime-tooltip {
        border-radius: 6px;
    }

    #watermark {
        position: fixed;
        bottom: 8px; right: 8px;
        pointer-events: none;
    }

    #watermark img {
        height: 32px;
        user-select: none;
        opacity: 0.4;
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
    import railwayLocationMarker from '../assets/images/railway_location_marker.svg';
    import airportLocationMarker from '../assets/images/airport_location_marker.svg';
    import selectedRailwayLocationMarker from '../assets/images/selected_railway_location_marker.svg';
    import selectedAirportLocationMarker from '../assets/images/selected_airport_location_marker.svg';

    const env = import.meta.env;
    
    type MenuKey = "railway" | "airport" | null;
    let menuLocationType = ref<MenuKey>(null);
    const selectedCrimeLocation = ref<string>("");
    const selectedCrimeLocationType = ref<'railway' | 'airport'>('railway');
    const selectedCrimes = ref<Crime[]>([]);
    const hoveredCrime = ref<CrimeTooltipData | null>(null);
    const hoveredLocationMarker = ref<{ id: string, name: string } | null>(null);
    const nearbyLocation = ref<{ id: string, name: string } | null>(null);
    const crimeTooltipPosition = ref({ x: 0, y: 0 });
    let clearSelectedLocation: (() => void) | null = null;
    let openNearbyLocationCrimePanel: () => void = () => {};
    const locationMarkerByType = {
        railway: "/markers/railway_marker.svg",
        airport: "/markers/airport_marker.svg"
    };
    const crimeMarkerByIconId: Record<number, string> = {
        0: "/markers/weapon_marker.svg",
        1: "/markers/drugs_marker.svg",
        2: "/markers/insult_marker.svg",
        3: "/markers/fist_marker.svg",
        4: "/markers/fraud_marker.svg",
        5: "/markers/theft_marker.svg",
        6: "/markers/unknown_marker.svg"
    };

    const railwayVisibility = ref<boolean>(true);
    const airportVisibility = ref<boolean>(true);

    interface Coordinate { lng: number, lat: number }
    interface Location { id: string, name: string, coordinates: Coordinate };
    interface Crime {
        id: number,
        location_id: string,
        coordinates: Coordinate,
        address: string,
        article: string,
        article_name: string,
        count: number,
        icon_id: number
    };

    type LocationCrimeLayers = { point: VectorLayer, label: VectorLayer };
    type CrimeTooltipData = Pick<Crime, 'article_name' | 'article' | 'count'>;

    const railwayList = ref<Location[]>(railwayResponse);
    const airportList = ref<Location[]>(airportResponse);
    let railwayCrimeMarkerLayers: LocationCrimeLayers | null = null;
    let airportCrimeMarkerLayers: LocationCrimeLayers | null = null;

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
        source: new VectorSource({
            features: createMarkers(
                railwayList.value,
                "/markers/railway_marker.svg",
                'point'
            )
        }),
        zIndex: 4
    });

    const airportMarkers = new VectorLayer({
        source: new VectorSource({
            features: createMarkers(
                airportList.value,
                "/markers/airport_marker.svg",
                'point'
            )
        }),
        zIndex: 5
    });

    const railwayLabels = new VectorLayer({
        source: new VectorSource({
            features: createMarkers(
                railwayList.value,
                "/markers/railway_marker.svg",
                'label'
            )
        }),
        zIndex: 6,
        minZoom: 10
    });

    const airportLabels = new VectorLayer({
        source: new VectorSource({
            features: createMarkers(
                airportList.value,
                "/markers/airport_marker.svg",
                'label'
            )
        }),
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

    function addLayer(
        source: object,
        style: Style,
        zIndex: number,
        renderBuffer: number = 100
    ): VectorLayer {
        return new VectorLayer({
            source: new VectorSource({
                features: new GeoJSON().readFeatures(source, {
                    dataProjection: 'EPSG:4326',
                    featureProjection: 'EPSG:3857',
                }),
            }),
            style,
            zIndex,
            renderBuffer,
        });
    }

    function openLocationList(key: MenuKey): void {
        if (menuLocationType.value === key) menuLocationType.value = null;
        else menuLocationType.value = key;
    }

    function closeLocationList(): void {
        menuLocationType.value = null;
    }

    function goToLocation(lng: number, lat: number, zoom: number = 16.5) {
        const view: View = map!.getView();
        view.setCenter(fromLonLat([lng, lat]));
        view.setZoom(zoom);
        closeCrimePanel();
    }

    type LabelOffset = { x: number, y: number }

    function createMarkers(
        list: Location[],
        iconSrc: string,
        locationType: 'point' | 'label',
        iconScale: number = 0.06,
        labelOffset: LabelOffset = { x: 0, y: 28 },
        labelFontSize: number = 14,
        textType: 'name' | 'count' = 'name',
        data: Crime[] | null = null
    ): Feature[] {
        return list.map(point => {
            const locationCrimes = data?.filter(crime =>
                point.id.includes(crime.location_id)
            ) ?? [];

            if (data && locationCrimes.length === 0) {
                return new Feature();
            }

            const feature = new Feature({
                geometry: new Point(
                    fromLonLat([
                        point.coordinates.lng,
                        point.coordinates.lat
                    ])
                )
            });

            let style: Style;

            const count = locationCrimes.reduce((sum, crime) => sum + crime.count, 0);

            switch (locationType) {
                case 'point':
                    style = new Style({
                        image: new Icon({
                            src: iconSrc,
                            scale: iconScale,
                            anchor: [0.5, 0.5]
                        })
                    });
                    break;
                case 'label':
                    style = new Style({
                        text: new Text({
                            text: textType === "name" || !data ? point.name : String(count),
                            offsetX: labelOffset.x,
                            offsetY: labelOffset.y,
                            font: `bold ${labelFontSize}px sans-serif`,
                            fill: new Fill({ color: 'white' }),
                            stroke: new Stroke({ color: 'black', width: 4 })
                        })
                    });
                    break;
            }

            feature.set("id", point.id);
            feature.set("name", point.name);
            feature.set("coordinates", point.coordinates);
            feature.setStyle(style);

            return feature;
        });
    }

    function createCrimeMarkers(
        list: Crime[],
        iconSrc: string,
        pointType: 'point' | 'label',
        iconScale: number = 0.06,
        labelOffset: LabelOffset = { x: 16, y: 16 },
        labelFontSize: number = 12,
        labelText: string | null = null
    ): Feature[] {
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
                        image: new Icon({
                            src: iconSrc,
                            scale: iconScale,
                            anchor: [0.5, 0.5]
                        }),
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
                            stroke: new Stroke({ color: 'white', width: 6 })
                        })
                    });
                    break;
            }

            feature.set("id", point.id);
            feature.set("name", point.article);
            feature.set("location_id", point.location_id);
            feature.set("address", point.address);
            feature.set("article", point.article);
            feature.set("article_name", point.article_name);
            feature.set("coordinates", point.coordinates);
            feature.set("count", point.count);
            feature.set("icon_id", point.icon_id);
            feature.setStyle(style);

            return feature;
        });
    }

    function createCrimeLayerPair(
        data: Crime[],
        iconSrc: string
    ): { point: VectorLayer, label: VectorLayer } {
        return {
            point: new VectorLayer({
                source: new VectorSource({
                    features: createCrimeMarkers(data, iconSrc, 'point')
                }),
                zIndex: 7,
                minZoom: 14
            }),
            label: new VectorLayer({
                source: new VectorSource({
                    features: createCrimeMarkers(data, iconSrc, 'label')
                }),
                zIndex: 7,
                minZoom: 16
            })
        };
    }

    function createLocationCrimeLayerPair(
        locations: Location[],
        data: Crime[],
        zIndex: number,
        markerSrc: string
    ): { point: VectorLayer, label: VectorLayer } {
        const createFeatures = (locationType: 'point' | 'label') =>
            createMarkers(
                locations,
                markerSrc,
                locationType,
                0.07,
                locationType === 'point' ? { x: 0, y: 0 } : { x: 18.5, y: -18.5 },
                14,
                locationType === 'point' ? 'name' : 'count',
                data
            );

        return {
            point: new VectorLayer({
                source: new VectorSource({
                    features: createFeatures('point')
                }),
                zIndex,
                minZoom: 1,
                maxZoom: 14
            }),
            label: new VectorLayer({
                source: new VectorSource({
                    features: createFeatures('label')
                }),
                zIndex,
                minZoom: 1,
                maxZoom: 14
            })
        };
    }

    function setLocationSelected(
        layers: { point: VectorLayer, label: VectorLayer },
        locationId: string,
        selected: boolean
    ): void {
        const pointFeature = layers.point.getSource()?.getFeatures().find(
            feature => feature.get("id") === locationId
        );
        const pointStyle = pointFeature?.getStyle();
        const marker = pointStyle instanceof Style ? pointStyle.getImage() : null;

        if (marker instanceof Icon) {
            const isAirport = locationId.includes("airport");
            marker.setSrc(
                selected
                    ? isAirport ? selectedAirportLocationMarker : selectedRailwayLocationMarker
                    : isAirport ? airportLocationMarker : railwayLocationMarker
            );
        }
        pointFeature?.changed();

        const labelFeature = layers.label.getSource()?.getFeatures().find(
            feature => feature.get("id") === locationId
        );
        const labelStyle = labelFeature?.getStyle();
        const text = labelStyle instanceof Style ? labelStyle.getText() : null;

        if (text) {
            text.setScale(selected ? 1.3 : 1.0);
            text.setOffsetX(selected ? 21.5 : 18.5);
            text.setOffsetY(selected ? -21.5 : -18.5);
        }
        labelFeature?.changed();
        layers.point.changed();
        layers.label.changed();
    }

    function closeCrimePanel(): void {
        clearSelectedLocation?.();
        selectedCrimeLocation.value = "";
        selectedCrimes.value = [];
    }

    function changeVisibility(type: 'railway' | 'airport'): void {
        switch (type) {
            case 'railway':
                railwayVisibility.value = !railwayVisibility.value;
                railwayMarkers.setVisible(railwayVisibility.value);
                railwayLabels.setVisible(railwayVisibility.value);
                railwayCrimeMarkerLayers?.point.setVisible(railwayVisibility.value);
                railwayCrimeMarkerLayers?.label.setVisible(railwayVisibility.value);
                if (!railwayVisibility.value && nearbyLocation.value?.id.includes("railway")) {
                    nearbyLocation.value = null;
                }
                if (menuLocationType.value === 'railway') menuLocationType.value = null;
                break;
            case 'airport':
                airportVisibility.value = !airportVisibility.value;
                airportMarkers.setVisible(airportVisibility.value);
                airportLabels.setVisible(airportVisibility.value);
                airportCrimeMarkerLayers?.point.setVisible(airportVisibility.value);
                airportCrimeMarkerLayers?.label.setVisible(airportVisibility.value);
                if (!airportVisibility.value && nearbyLocation.value?.id.includes("airport")) {
                    nearbyLocation.value = null;
                }
                if (menuLocationType.value === 'airport') menuLocationType.value = null;
                break;
        }
    }

    async function getData(): Promise<Crime[]> {
        const response = await fetch(env.VITE_BACKEND_DATA_URL);

        const data: Crime[] = await response.json();
        return data;
    }

    onMounted(async () => {
        const alldata = await getData();
        const weaponData = alldata.filter(point => point.icon_id === 0);
        const drugData = alldata.filter(point => point.icon_id === 1);
        const fistData = alldata.filter(point => point.icon_id === 3);
        const fraudData = alldata.filter(point => point.icon_id === 4);
        const theftData = alldata.filter(point => point.icon_id === 5);
        const unknownData = alldata.filter(point => point.icon_id === 6);

        const railwayData = alldata.filter(point => point.location_id.includes("railway"));
        const airportData = alldata.filter(point => point.location_id.includes("airport"));

        const railwayCrimeMarkers = createLocationCrimeLayerPair(
            railwayList.value,
            railwayData,
            8,
            railwayLocationMarker
        );
        const airportCrimeMarkers = createLocationCrimeLayerPair(
            airportList.value,
            airportData,
            9,
            airportLocationMarker
        );
        railwayCrimeMarkerLayers = railwayCrimeMarkers;
        airportCrimeMarkerLayers = airportCrimeMarkers;
        railwayCrimeMarkers.point.setVisible(railwayVisibility.value);
        railwayCrimeMarkers.label.setVisible(railwayVisibility.value);
        airportCrimeMarkers.point.setVisible(airportVisibility.value);
        airportCrimeMarkers.label.setVisible(airportVisibility.value);

        const crimeLayers = {
            weapon: createCrimeLayerPair(weaponData, "/markers/weapon_marker.svg"),
            drug: createCrimeLayerPair(drugData, "/markers/drugs_marker.svg"),
            fist: createCrimeLayerPair(fistData, "/markers/fist_marker.svg"),
            fraud: createCrimeLayerPair(fraudData, "/markers/fraud_marker.svg"),
            theft: createCrimeLayerPair(theftData, "/markers/theft_marker.svg"),
            unknown: createCrimeLayerPair(unknownData, "/markers/unknown_marker.svg")
        };
        const crimeMarkerLayers = Object.values(crimeLayers).flatMap(
            ({ point, label }) => [point, label]
        );

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
                crimeLayers.unknown.point,
                crimeLayers.unknown.label,
                crimeLayers.fist.point,
                crimeLayers.fist.label,
                crimeLayers.fraud.point,
                crimeLayers.theft.point,
                crimeLayers.theft.label,
                crimeLayers.fraud.label,
                crimeLayers.drug.point,
                crimeLayers.drug.label,
                crimeLayers.weapon.point,
                crimeLayers.weapon.label,
                railwayCrimeMarkers.point,
                railwayCrimeMarkers.label,
                airportCrimeMarkers.point,
                airportCrimeMarkers.label
            ],
            view: new View({
                center: fromLonLat([66.3, 48.5]),
                zoom: 5,
                maxZoom: 18,
                extent: [
                    ...fromLonLat([42.2, 34.0]),
                    ...fromLonLat([92.5, 58.5])
                ],
            }),
            interactions: defaultInteractions({ doubleClickZoom: false })
        });

        let selectedLocationId: string | null = null;
        clearSelectedLocation = () => {
            if (!selectedLocationId) return;

            const selectedLayers = selectedLocationId.includes("airport")
                ? airportCrimeMarkers
                : railwayCrimeMarkers;
            setLocationSelected(selectedLayers, selectedLocationId, false);
            selectedLocationId = null;
        };

        map.on('dblclick', (event) => {
            event.preventDefault();
            const view = map!.getView();
            const feature = map?.forEachFeatureAtPixel(
                event.pixel,
                (candidate, layer) => {
                    if (
                        layer === railwayCrimeMarkers.point ||
                        layer === railwayCrimeMarkers.label ||
                        layer === airportCrimeMarkers.point ||
                        layer === airportCrimeMarkers.label
                    ) {
                        return candidate;
                    }
                },
                { hitTolerance: 1 }
            );

            const geometry = feature?.getGeometry();
            if (geometry instanceof Point) {
                view.animate({
                    center: geometry.getCoordinates(),
                    zoom: 16.5,
                    duration: 750
                });
            } else {
                view.setCenter(fromLonLat([66.3, 48.5]));
                view.setZoom(1.0);
            }
        });

        map.on('singleclick', (event) => {
            const feature = map?.forEachFeatureAtPixel(
                event.pixel,
                (candidate, layer) => {
                    if (
                        layer === railwayCrimeMarkers.point ||
                        layer === railwayCrimeMarkers.label ||
                        layer === airportCrimeMarkers.point ||
                        layer === airportCrimeMarkers.label
                    ) {
                        return candidate;
                    }
                }, { hitTolerance: 1 }
            );

            if (!feature) {
                closeCrimePanel();
                closeLocationList();
                return;
            }

            const locationId = feature.get("id");
            const locationName = feature.get("name");

            if (typeof locationId !== "string" || typeof locationName !== "string") {
                return;
            }

            openCrimePanelForLocation(locationId, locationName);
        });

        function openCrimePanelForLocation(locationId: string, locationName: string): void {
            if (selectedLocationId && selectedLocationId !== locationId) {
                const previousLayers = selectedLocationId.includes("airport")
                    ? airportCrimeMarkers
                    : railwayCrimeMarkers;
                setLocationSelected(previousLayers, selectedLocationId, false);
            }

            const selectedLayers = locationId.includes("airport")
                ? airportCrimeMarkers
                : railwayCrimeMarkers;
            setLocationSelected(selectedLayers, locationId, true);
            selectedLocationId = locationId;

            selectedCrimeLocation.value = locationName;
            selectedCrimeLocationType.value = locationId.includes("railway")
                ? "railway"
                : "airport";
            selectedCrimes.value = alldata.filter(crime =>
                locationId.includes(crime.location_id)
            );
        }

        openNearbyLocationCrimePanel = () => {
            const location = nearbyLocation.value;
            if (location) {
                openCrimePanelForLocation(location.id, location.name);
            }
        };

        function updateNearbyLocation(): void {
            const zoom = map!.getView().getZoom() ?? 0;
            if (zoom < 14) {
                nearbyLocation.value = null;
                return;
            }

            const size = map!.getSize();
            if (!size) {
                nearbyLocation.value = null;
                return;
            }

            const centerPixel = [size[0] / 2, size[1] / 2];
            const candidates = [
                { layers: railwayCrimeMarkers, visible: railwayVisibility.value },
                { layers: airportCrimeMarkers, visible: airportVisibility.value }
            ];
            let nearest: { id: string, name: string } | null = null;
            let nearestDistance = 640;

            for (const { layers, visible } of candidates) {
                if (!visible) continue;

                for (const layer of [layers.point, layers.label]) {
                    for (const feature of layer.getSource()?.getFeatures() ?? []) {
                        const id = feature.get("id");
                        const name = feature.get("name");
                        const geometry = feature.getGeometry();

                        if (
                            typeof id !== "string" ||
                            typeof name !== "string" ||
                            !(geometry instanceof Point)
                        ) {
                            continue;
                        }

                        const pixel = map!.getPixelFromCoordinate(geometry.getCoordinates());
                        if (!pixel) continue;

                        const distance = Math.hypot(
                            pixel[0] - centerPixel[0],
                            pixel[1] - centerPixel[1]
                        );

                        if (distance <= nearestDistance) {
                            nearestDistance = distance;
                            nearest = { id, name };
                        }
                    }
                }
            }

            nearbyLocation.value = nearest;
        }

        map.on('moveend', updateNearbyLocation);
        map.getView().on('change:resolution', updateNearbyLocation);

        map.on('pointermove', (event) => {
            let isLocationMarker = false;
            let locationAtPointer: { id: string, name: string } | null = null;
            let crimeAtPointer: CrimeTooltipData | null = null;

            map?.forEachFeatureAtPixel(
                event.pixel,
                (feature, layer) => {
                    if (layer === railwayCrimeMarkers.point ||
                        layer === railwayCrimeMarkers.label ||
                        layer === airportCrimeMarkers.point ||
                        layer === airportCrimeMarkers.label) {
                        isLocationMarker = true;
                        const id = feature.get("id");
                        const name = feature.get("name");

                        if (typeof id === "string" && typeof name === "string") {
                            locationAtPointer = { id, name };
                            return true;
                        }

                        return undefined;
                    }

                    if (!layer || !crimeMarkerLayers.includes(layer as VectorLayer)) {
                        return undefined;
                    }

                    const article = feature.get("article");
                    const articleName = feature.get("article_name");
                    const count = feature.get("count");

                    if (
                        typeof article === "string" &&
                        typeof articleName === "string" &&
                        typeof count === "number"
                    ) {
                        crimeAtPointer = {
                            article,
                            article_name: articleName,
                            count
                        };
                        return true;
                    }

                    return undefined;
                },
                { hitTolerance: 1 }
            );

            hoveredLocationMarker.value = locationAtPointer;
            hoveredCrime.value = crimeAtPointer;
            if (crimeAtPointer || locationAtPointer) {
                const mapBounds = map!.getTargetElement().getBoundingClientRect();
                crimeTooltipPosition.value = {
                    x: mapBounds.left + event.pixel[0] + 12,
                    y: mapBounds.top + event.pixel[1] + 12
                };
            }

            map!.getTargetElement().style.cursor = isLocationMarker
                ? 'pointer'
                : '';
        });
    });

    onUnmounted(() => map?.setTarget(undefined));
</script>