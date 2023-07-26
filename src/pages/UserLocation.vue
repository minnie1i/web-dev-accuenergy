<template>
    <div class="App">
        <!-- <section class="ui two column left grid"> -->
        <div class="ui grid left">
            <div class="four wide column">
                <form class="ui segment large form">
                    <div class="ui message red" v-show="error">{{ error }}</div>
                    <div class="ui segment">
                        <div class="field">
                            <div class="ui right icon input large" :class="{loading:spinner}">
                                <input type="text" placeholder="Search here!" v-model="inputAddress" @keyup.enter.exact.stop="searchInputAddress"/>
                                <i class="location arrow link icon" @click="locatorButtonPressed"></i>
                            </div>
                        </div>
                        <button class="ui button" @click.stop="searchInputAddress">Show On Map</button>
                    </div>
                    <div class="ui message" v-show="timeZone">The current time of lastest searched location is {{ localTime }} ({{ timeZone }})</div>
                </form>
                <div class="ui segment">
                    <div class="pagination">
                        <button class="ui button blue" @click="prevPage">Previous</button>
                        <button class="ui button blue" @click="nextPage">Next</button>
                        <button class="ui button maroon" @click="deleteSelected">Delete</button>
                    </div>
                    <table class="ui very basic table">
                        <thead>
                            <th></th>
                            <th>Address</th>
                        </thead>
                        <tbody>
                            <tr v-for="(location, index) in paginatedLocations" :key="index">
                                <td><input type="checkbox" v-model="selectedPlaces" :value="location.id"></td>
                                <td>{{ location.address }}</td>
                            </tr>
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
        <div id="map"></div>
    </div>
</template>

<script>
import axios from 'axios'
export default{
    data() {
        return {
            API_KEY: "",
            map: "",
            inputAddress: "",
            selectedPlace: {},
            currentPlace: {},
            error:"",
            spinner: false,
            loading: false,
            pageSize: 10,
            currentPage: 0,
            selectAllChecked: false,
            selectedPlaces: [],
            markers: [],
            locations: [],
            timeZone: '',
            localTime: '',
            displayTimer: 0,
        }
    },
    computed: {
        paginatedLocations() {
            const start = this.currentPage * this.pageSize;
            const end = start + this.pageSize;
            return this.locations.slice(start, end);
        },
    },
    mounted() {
        this.map = new google.maps.Map(document.getElementById("map"), {
            zoom: 15,
            center: new google.maps.LatLng(43.775007970056635, -79.3260265108113),
            mapTypeId: google.maps.MapTypeId.ROADMAP
        })
    },
    methods: {
        prevPage() {
            if (this.currentPage > 0) {
            this.currentPage--;
            }
        },
        nextPage() {
            const maxPage = Math.floor(this.locations.length / this.pageSize);
            if (this.currentPage < maxPage) {
            this.currentPage++;
            }
        },
        deleteSelected() {
            console.log(this.selectedPlaces)
            this.locations.forEach(location => {
                if (this.selectedPlaces.includes(location.id)) {
                    location.marker.setMap(null)
                }
            })
            this.locations = this.locations.filter(
                (location) => !this.selectedPlaces.includes(location.id)
            );
            this.selectedPlaces = [];
            this.selectAllChecked = false
        },
        locatorButtonPressed() {
            this.spinner = true;
            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition(
                    position => {
                        // console.log(position.coords.latitude, position.coords.longitude);
                        this.getAddressFrom(position.coords.latitude, position.coords.longitude);
                        this.showUserLocationOnTheMap(position.coords.latitude, position.coords.longitude)
                    },
                    error => {
                        this.error = "We are unable to find your address! Please enter manually.";
                        this.spinner = false;
                        // console.log(error.message);
                    }
                );
            } else {
                this.error = "geolocation api not supported";
                this.spinner = false;
            }
        },
        getAddressFrom(lat, long) {
            axios.get("https://maps.googleapis.com/maps/api/geocode/json?latlng=" + lat + "," + long + "&key=" + this.API_KEY)
            .then(response => {
                if (response.data.error_message){
                    // console.log(response.data.error_message)
                    this.error = response.data.error_message;
                } else {
                    // console.log(response.data.results[0].formatted_address)
                    this.selectedPlace = response.data.results[0]
                    this.inputAddress = response.data.results[0].formatted_address;
                }
                this.spinner = false;
            })
            .catch(error => {
                this.error = error.message;
                this.spinner = false;
                // console.log(error.message)
            })
        },
        searchInputAddress() {
            if (!this.loading) {
                this.loading = true
                clearInterval(this.displayTimer)
                this.error=""
                this.timeZone = ""
                this.localTime = ''
                console.log("inputAddress",this.inputAddress)
                var request = {
                    query: this.inputAddress,
                    fields: ["formatted_address", "geometry"],
                };
                console.log("request", request)
                var service = new google.maps.places.PlacesService(this.map);
                service.findPlaceFromQuery(request, (results, status) => {
                    console.log("status", status)
                    console.log("result", results)
                    if (status === google.maps.places.PlacesServiceStatus.OK && results) {
                        this.selectedPlace = results[0]
                        var position = {lat: this.selectedPlace.geometry.location.lat(), lng: this.selectedPlace.geometry.location.lng()}
                        this.map.setCenter(this.selectedPlace.geometry.location);
                        var newMarker = new google.maps.Marker({
                            position: new google.maps.LatLng(position.lat, position.lng),
                            map: this.map
                        })
                        this.markers.push(newMarker)
                        this.locations.push({
                            id: this.locations.length + 1,
                            address: this.selectedPlace.formatted_address,
                            lat: position.lat,
                            lng: position.lng,
                            marker: newMarker
                        });
                        this.getLocalTimezoneOffset(position.lat, position.lng)
                    } else {
                        this.error = status
                    }
                    this.loading = false
                });
            }
        },
        getLocalTimezoneOffset(lat, long) {
            var targetDate = new Date()
            var timestamp = targetDate.getTime()/1000 + targetDate.getTimezoneOffset() * 60
            axios.get("https://maps.googleapis.com/maps/api/timezone/json?location=" + lat + "%2C" + long + "&timestamp=" + timestamp + "&key=" + this.API_KEY)
            .then(response => {
                if (response.data.error_message){
                    // console.log(response.data.error_message)
                    this.error = response.data.error_message;
                } else {
                    console.log(response.data)
                    if (response.data.status == 'OK') {
                        var offsets = response.data.dstOffset * 1000 + response.data.rawOffset * 1000
                        var localdate = new Date(timestamp * 1000 + offsets)
                        var refreshDate = new Date()
                        var millisecondselapsed = refreshDate - targetDate
                        localdate.setMilliseconds(localdate.getMilliseconds()+ millisecondselapsed)
                        this.displayTimer = setInterval(() => {
                            localdate.setSeconds(localdate.getSeconds() + 1)
                            this.localTime = localdate.toLocaleTimeString()
                        }, 1000)
                        this.timeZone = response.data.timeZoneName
                    }
                }
            })
            .catch(error => {
                this.error = error.message;
                // console.log(error.message)
            })
        }
    }
}
</script>

<style>
/* .ui.two.column.centered.grid {
    margin-left: 10px;
} */
.App {
    height: 100%;
}
.ui.grid.left {
    height: 100%;
}
.four.wide.column {
    z-index: 1;
    height: 100%;
}
.ui.button,
.location.arrow.icon {
    background-color: maroon;
    color: white;
}
.pac-item {
    padding: 10px;
    font-size: 14px;
    cursor: pointer;
}
.pac-item:hover {
    background-color: maroon;
}
.pac-item-query {
    font-size: 16px;
    padding-right: 8px;
}
.pac-item:hover .pac-item-query {
    color: white;
}
.pac-icon {
    display: none;
}
#map {
    position: absolute;
    top: 0;
    left: 0;
    bottom: 0;
    right: 0;
    background: maroon;
}
</style>