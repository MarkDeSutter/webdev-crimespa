<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('');
let dialog_err = ref(false);

let incidents = ref([]);
let neighborhoods = ref([]);
let crime_counts = reactive({});
let map = reactive(
    {
        leaflet: null,
        center: {
            lat: 44.955139,
            lng: -93.102222,
            address: ''
        },
        zoom: 12,
        bounds: {
            nw: {lat: 45.008206, lng: -93.217977},
            se: {lat: 44.883658, lng: -92.993787}
        },
        neighborhood_markers: [
            {location: [44.942068, -93.020521], marker: null, name: "Conway/Battlecreek/Highwood"},
            {location: [44.977413, -93.025156], marker: null, name: "Greater East Side"},
            {location: [44.931244, -93.079578], marker: null, name: "West Side"},
            {location: [44.956192, -93.060189], marker: null, name: "Dayton's Bluff"},
            {location: [44.978883, -93.068163], marker: null, name: "Payne/Phalen"},
            {location: [44.975766, -93.113887], marker: null, name: "North End"},
            {location: [44.959639, -93.121271], marker: null, name: "Thomas/Dale(Frogtown)"},
            {location: [44.947700, -93.128505], marker: null, name: "Summit/University"},
            {location: [44.930276, -93.119911], marker: null, name: "West Seventh"},
            {location: [44.982752, -93.147910], marker: null, name: "Como"},
            {location: [44.963631, -93.167548], marker: null, name: "Hamline/Midway"},
            {location: [44.973971, -93.197965], marker: null, name: "St. Anthony"},
            {location: [44.949043, -93.178261], marker: null, name: "Union Park"},
            {location: [44.934848, -93.176736], marker: null, name: "Macalester-Groveland"},
            {location: [44.913106, -93.170779], marker: null, name: "Highland"},
            {location: [44.937705, -93.136997], marker: null, name: "Summit Hill"},
            {location: [44.949203, -93.093739], marker: null, name: "Capitol River"}
        ]
    }
);

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
    // Create Leaflet map (set bounds and valied zoom levels)
    map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
        attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
        minZoom: 11,
        maxZoom: 18
    }).addTo(map.leaflet);
    map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);

    // Get boundaries for St. Paul neighborhoods
    let district_boundary = new L.geoJson();
    district_boundary.addTo(map.leaflet);
    fetch('data/StPaulDistrictCouncil.geojson')
    .then((response) => {
        return response.json();
    })
    .then((result) => {
        result.features.forEach((value) => {
            district_boundary.addData(value);
        });
    })
    .catch((error) => {
        console.log('Error:', error);
    });
});


// FUNCTIONS
// Function called once user has entered REST API URL
async function initializeCrimes() {
    // TODO: get code and neighborhood data
    //       get initial 1000 crimes

    console.log("Crime URL:" + crime_url.value);

    //uses await to asynchronously build 2 different arrays, one for the incidents and the other for the neighborhood names
    try {
        let incidents_response = await fetch(crime_url.value + '/incidents', { method: 'GET' });
        let incident_data = await incidents_response.json();
        console.log(incident_data);
        incidents.value = incident_data;

        let neighborhood_names_map = {};
        let neighborhood_response = await fetch(crime_url.value + '/neighborhoods', { method: 'GET'});
        let neighborhood_data = await neighborhood_response.json();
        for(let i = 0; i < neighborhood_data.neighborhoods.length; i++){
            let id = neighborhood_data.neighborhoods[i].neighborhood_number;
            let name = neighborhood_data.neighborhoods[i].neighborhood_name;
            neighborhood_names_map[id] = name;
        }
        //console.log(neighborhood_names_map);

        if(incidents.value.length > 0){
            for(let i = 0; i < incidents.value.length; i++){
                incidents.value[i].neighborhood = neighborhood_names_map[incidents.value[i].neighborhood_number];
                if(!(incidents.value[i].neighborhood in crime_counts)){
                    crime_counts[incidents.value[i].neighborhood] = 1;
                }
                else{
                    crime_counts[incidents.value[i].neighborhood] = crime_counts[incidents.value[i].neighborhood] + 1;
                }
            }
            
            console.log(crime_counts);
            updateMarkerTitles();
        }

        console.log(incidents);

    }
    catch (err) {
        console.error('Failed to load incidents or neighborhoods:', err);
    }
}

// Update marker titles with crime counts after incidents are loaded
function updateMarkerTitles() {
    for(let i = 0; i < map.neighborhood_markers.length; i++){
        let name = map.neighborhood_markers[i].name;
        let crime_count = crime_counts[name];
        map.neighborhood_markers[i].marker = L.marker(map.neighborhood_markers[i].location, { title: name  + '\nCrimes: ' + crime_count}).addTo(map.leaflet);
        map.neighborhood_markers[i].marker.bindPopup(name);
    }
}


// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
    }
}
</script>

<template>
    <dialog id="rest-dialog" open>
        <h1 class="dialog-header">St. Paul Crime REST API</h1>
        <label class="dialog-label">URL: </label>
        <input id="dialog-url" class="dialog-input" type="url" v-model="crime_url" placeholder="http://localhost:8000" />
        <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
        <br/>
        <button class="button" type="button" @click="closeDialog">OK</button>
    </dialog>
    <div class="grid-container ">
        <div class="grid-x grid-padding-x">
            <div id="leafletmap" class="cell auto"></div>
        </div>
    </div>

    <div>
        <table>
            <thead>
                <tr>
                    <td>Case Number</td>
                    <td>Date</td>
                    <td>Incident</td>
                    <td>Police Grid #</td>
                    <td>Neighborhood</td>
                    <td>Block</td>
                </tr>
            </thead>
            <tbody>
                <tr v-for="(incident, index) in incidents">
                    <td>{{ incident.case_number }}</td>
                    <td>{{ incident.date_time }}</td>
                    <td>{{ incident.incident }}</td>
                    <td>{{ incident.police_grid }}</td>
                    <td>{{ incident.neighborhood }}</td>
                    <td>{{ incident.block }}</td>
                </tr>
            </tbody>
        </table>
    </div>
</template>

<style scoped>
#rest-dialog {
    width: 20rem;
    margin-top: 1rem;
    z-index: 1000;
}

#leafletmap {
    height: 500px;
}

.dialog-header {
    font-size: 1.2rem;
    font-weight: bold;
}

.dialog-label {
    font-size: 1rem;
}

.dialog-input {
    font-size: 1rem;
    width: 100%;
}

.dialog-error {
    font-size: 1rem;
    color: #D32323;
}
</style>
