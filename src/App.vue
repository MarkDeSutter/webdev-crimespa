<script setup>
import { reactive, ref, onMounted } from 'vue'

let crime_url = ref('');
let dialog_err = ref(false);

let neighborhood_names_map = {};

let incidents = ref([]);
let neighborhoods = ref([]);
let crime_counts = reactive({});
let mapBounds = reactive({
    minlat: 0,
    maxlat: 0,
    minlon: 0,
    maxlon: 0
});
let visible_neighborhoods = ref([]);

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
            {location: [44.942068, -93.020521], marker: null, id: 1, name: "Conway/Battlecreek/Highwood"},
            {location: [44.977413, -93.025156], marker: null, id: 2, name: "Greater East Side"},
            {location: [44.931244, -93.079578], marker: null, id: 3, name: "West Side"},
            {location: [44.956192, -93.060189], marker: null, id: 4, name: "Dayton's Bluff"},
            {location: [44.978883, -93.068163], marker: null, id: 5, name: "Payne/Phalen"},
            {location: [44.975766, -93.113887], marker: null, id: 6, name: "North End"},
            {location: [44.959639, -93.121271], marker: null, id: 7, name: "Thomas/Dale(Frogtown)"},
            {location: [44.947700, -93.128505], marker: null, id: 8, name: "Summit/University"},
            {location: [44.930276, -93.119911], marker: null, id: 9, name: "West Seventh"},
            {location: [44.982752, -93.147910], marker: null, id: 10, name: "Como"},
            {location: [44.963631, -93.167548], marker: null, id: 11, name: "Hamline/Midway"},
            {location: [44.973971, -93.197965], marker: null, id: 12, name: "St. Anthony"},
            {location: [44.949043, -93.178261], marker: null, id: 13, name: "Union Park"},
            {location: [44.934848, -93.176736], marker: null, id: 14, name: "Macalester-Groveland"},
            {location: [44.913106, -93.170779], marker: null, id: 15, name: "Highland"},
            {location: [44.937705, -93.136997], marker: null, id: 16, name: "Summit Hill"},
            {location: [44.949203, -93.093739], marker: null, id: 17, name: "Capitol River"}
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

    //updates table based on neighborhoods in view
    map.leaflet.on('moveend', () => {        
        let bounds = map.leaflet.getBounds();
        mapBounds.minlat = bounds._southWest.lat;
        mapBounds.maxlat = bounds._northEast.lat;
        mapBounds.minlon = bounds._southWest.lng;
        mapBounds.maxlon = bounds._northEast.lng;
        //console.log(mapBounds);

        visible_neighborhoods.value = [];
        incidents.value = [];
        
        //checks to see what neighborhoods are currently visible
        for(let neighborhood of map.neighborhood_markers){
            let lat = neighborhood.location[0];
            let lon = neighborhood.location[1];
            if(lat >= mapBounds.minlat && lat <= mapBounds.maxlat && lon >= mapBounds.minlon && lon <= mapBounds.maxlon){
                visible_neighborhoods.value.push(neighborhood.id);
            }
        }
        //console.log(visible_neighborhoods);
        initializeCrimes();
    });
});


// FUNCTIONS
// Function called once user has entered REST API URL
async function initializeCrimes() {
    // TODO: get code and neighborhood data
    //       get initial 1000 crimes

    //console.log("Crime URL:" + crime_url.value);

    //adds top 1000 incidents to incidents array based on visible neighborhoods
    try {
        //checks to see what neighborhoods are visible
        let ids = '';
        for(let id of visible_neighborhoods.value){
            ids += `${id},`;
        }
        if(ids.length != 0){
            ids = ids.substring(0, ids.length - 1);
        }
        //console.log(ids);

        //fetches incidents and puts them in incidents array
        let incidents_response = await fetch(crime_url.value + '/incidents?neighborhood=' + ids, { method: 'GET' });
        let incident_data = await incidents_response.json();
        //console.log(incident_data);
        incidents.value = incident_data;

        

        if(incidents.value.length > 0){
            crime_counts = {};
            for(let i = 0; i < incidents.value.length; i++){
                incidents.value[i].neighborhood = neighborhood_names_map[incidents.value[i].neighborhood_number];
                if(incidents.value[i].code <= 863 && incidents.value[i].code != 614){
                    incidents.value[i].color = "red";
                }
                else if(incidents.value[i].code > 863 && incidents.value[i].code <= 1463){
                    incidents.value[i].color = "blue";
                }
                else{
                    incidents.value[i].color = "green";
                }
                //counts crimes
                if(!(incidents.value[i].neighborhood in crime_counts)){
                    crime_counts[incidents.value[i].neighborhood] = 1;
                }
                else{
                    crime_counts[incidents.value[i].neighborhood] = crime_counts[incidents.value[i].neighborhood] + 1;
                }
            }
            
            //console.log(crime_counts);
            updateMarkerTitles();
        }

        //console.log(incidents);

    }
    catch (err) {
        console.error('Failed to load incidents or neighborhoods:', err);
    }
}

//builds map of neighborhoods ids : names
async function buildNeighborhoodMap(){
        let neighborhood_response = await fetch(crime_url.value + '/neighborhoods', { method: 'GET'});
        let neighborhood_data = await neighborhood_response.json();
        for(let i = 0; i < neighborhood_data.length; i++){
            let id = neighborhood_data[i].neighborhood_number;
            let name = neighborhood_data[i].neighborhood_name;
            neighborhood_names_map[id] = name;
        }
        //console.log(neighborhood_names_map);
}
// Update marker titles with crime counts after incidents are loaded
function updateMarkerTitles() {
    for(let i = 0; i < map.neighborhood_markers.length; i++){
        let name = map.neighborhood_markers[i].name;
        let crime_count = crime_counts[name];
        map.neighborhood_markers[i].marker = L.marker(map.neighborhood_markers[i].location).addTo(map.leaflet);
        map.neighborhood_markers[i].marker.bindPopup("<b>" + name + "</b><br><b>Crimes: " + crime_count + "</b>");
    }
}

//Creates a marker on the map for an incident
function createIncidentMarker(incident) {
    let lat, lon;
    let block = incident.block;
    let numbers_in_address = block.substring(0, block.indexOf(' '));
    numbers_in_address = numbers_in_address.replaceAll('X', '0');
    block = block.replace(block.substring(0, block.indexOf(' ')), numbers_in_address);
    console.log(block);

    fetch('https://nominatim.openstreetmap.org/search?q=' + block + ' St. Paul MN&format=jsonv2')
    .then((response) => {
        return response.json();
    })
    .then((data) => {
        console.log(data);
        lat = data[0].lat;
        lon = data[0].lon;

        console.log(`${lat} ${lon}`);

    // use a colored circle marker (color from incident.color if available)
    const marker = L.circleMarker([Number(lat), Number(lon)], {
        radius: 6,
        fillColor: incident.color || 'red',
        color: '#000',
        weight: 1,
        fillOpacity: 0.9,
        title: incident.incident
    }).addTo(map.leaflet);

    // popup with useful info
    marker.bindPopup(`<b>Date: ${incident.date}</b><br>
    <b>Time: ${incident.time}</b><br>
    <b>Incident: ${incident.incident}`).openPopup();

    // optionally keep reference so you can remove/update later
    incident._marker = marker

        })
    .catch((err) => {
        console.log(err);
    });
}

//removes incident marker
function removeIncidentMarker(incident){
    incident._marker.remove();
}

// Function called when user presses 'OK' on dialog box
function closeDialog() {
    let dialog = document.getElementById('rest-dialog');
    let url_input = document.getElementById('dialog-url');
    if (crime_url.value !== '' && url_input.checkValidity()) {
        dialog_err.value = false;
        dialog.close();
        buildNeighborhoodMap();
        initializeCrimes();
    }
    else {
        dialog_err.value = true;
    }
}

// PUT: Add New Incident
let newIncident = reactive({
    case_number: '',
    date: '',
    time: '',
    code: '',
    incident: '',
    police_grid: '',
    neighborhood_number: '',
    block: ''
});

let addError = ref('');
let addSuccess = ref('');

async function submitNewIncident() {
    addError.value = '';
    addSuccess.value = '';

    // check for missing fields
    for (let key in newIncident) {
        if (!newIncident[key]) {
            addError.value = "All fields are required.";
            return;
        }
    }

    // ensure date and time are clean 
    let dateOnly = newIncident.date.substring(0, 10); // YYYY-MM-DD
    let timeOnly = newIncident.time.substring(0, 8);  // HH:MM:SS
    if (timeOnly.length === 5) timeOnly += ":00"; // add seconds if missing

    try {
        let response = await fetch(crime_url.value + "/new-incident", {
            method: "PUT",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
                ...newIncident,
                date: dateOnly,
                time: timeOnly
            })
        });

        let txt = await response.text();

        if (!response.ok) {
            addError.value = txt;
            return;
        }

        addSuccess.value = "Incident added!";
        Object.keys(newIncident).forEach(k => newIncident[k] = "");

        initializeCrimes();
    }
    catch (err) {
        addError.value = "Network error";
    }
}




// DELETE: Remove Incident
async function deleteIncident(case_number) {
    try {
        let response = await fetch(crime_url.value + "/remove-incident", {
            method: "DELETE",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({ case_number })
        });

        let txt = await response.text();

        if (!response.ok) {
            alert(txt);
            return;
        }

        alert("Incident deleted");

        // Refresh table
        initializeCrimes();
    }
    catch (err) {
        alert("Network error");
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
    <h2>Add New Crime Incident</h2>

    <div class="incident-form">
        <p v-if="addError" style="color:red">{{ addError }}</p>
        <p v-if="addSuccess" style="color:green">{{ addSuccess }}</p>

        <input v-model="newIncident.case_number" placeholder="Case Number">
        <input v-model="newIncident.date" type="date">
        <input v-model="newIncident.time" type="time">
        <input v-model="newIncident.code" placeholder="Code">
        <input v-model="newIncident.incident" placeholder="Incident Description">
        <input v-model="newIncident.police_grid" placeholder="Police Grid #">
        <input v-model="newIncident.neighborhood_number" placeholder="Neighborhood #">
        <input v-model="newIncident.block" placeholder="Block Address">

        <button @click="submitNewIncident">Submit</button>
    </div>

    <div>
        <table>
            <thead>
                <tr>
                    <td>Key: </td>
                    <td style="background-color: red;">Violent Crimes</td>
                    <td style="background-color: blue;">Property Crimes</td>
                    <td style="background-color: green;">Other</td>
                </tr>
                <tr>
                    <td>Case Number</td>
                    <td>Date</td>
                    <td>Time</td>
                    <td>Incident</td>
                    <td>Police Grid #</td>
                    <td>Neighborhood</td>
                    <td>Block</td>
                </tr>
            </thead>
            <tbody>
                <tr v-for="(incident, index) in incidents" :style="{ backgroundColor: incident.color}">
                    <td>{{ incident.case_number }}</td>
                    <td>{{ incident.date }}</td>
                    <td>{{ incident.time }}</td>
                    <td>{{ incident.incident }}</td>
                    <td>{{ incident.police_grid }}</td>
                    <td>{{ incident.neighborhood }}</td>
                    <td>{{ incident.block }}</td>
                    <td><button @click="createIncidentMarker(incident)">Create marker</button></td>
                    <td><button @click="removeIncidentMarker(incident)">Delete Marker</button></td>
                    <td><button @click="deleteIncident(incident.case_number)">Delete from DB</button></td>
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
