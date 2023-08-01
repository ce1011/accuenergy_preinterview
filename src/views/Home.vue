<template>
  <div>
    <v-container>
      <v-row>
        <v-col cols="6">
          <v-row>
            <v-col>
              <v-btn @click="getCurrentLocation" color="primary"
                >Acquire current location</v-btn
              >
            </v-col>
          </v-row>
          <v-row align="center">
            <v-col align-self="center">
              <v-text-field
                label="Location"
                v-model="locationSearch"
                v-on:keyup.enter="getPlaceAutoComplete(locationSearch)"
              ></v-text-field>
            </v-col>
            <v-col cols="2" align-self="center">
              <v-btn
                @click="getPlaceAutoComplete(locationSearch)"
                color="primary"
                >Search</v-btn
              >
            </v-col>
          </v-row>
          <v-row align="center">
            <v-col
              ><p>
                Timezone and local time of latest search: <br />{{
                  timezoneLocalTime
                }}
              </p></v-col
            >
          </v-row>

          <v-row align="center">
            <v-col>
              <p>Current Location: <br />{{ currentLocationAddress }}</p>
            </v-col>
          </v-row>
          <v-row>
            <v-col>
              <GMapMap
                ref="googlemap"
                :center="center"
                :zoom="11"
                map-type-id="terrain"
                style="width: 100%; height: 300px"
                id="map"
              >
                <GMapMarker
                  :key="index"
                  v-for="(m, index) in markerList"
                  :position="m.latlng"
                />
              </GMapMap>
            </v-col>
          </v-row>
        </v-col>
        <v-col cols="6">
          <v-data-table
            v-model="selectedMarker"
            items-per-page="10"
            :headers="headers"
            show-select
            :items="markerList"
            item-value="placeId"
            class="elevation-1"
          >
            <template v-slot:top>
              <v-toolbar flat>
                <v-toolbar-title>History</v-toolbar-title>

                <v-btn @click="deleteMarker" color="primary" dark class="mb-2">
                  Delete
                </v-btn>
              </v-toolbar>
            </template>
          </v-data-table>
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script lang="ts" setup>
import { ref } from "vue";
import { VDataTable } from "vuetify/labs/VDataTable";
import { Marker } from "../models/marker";
import moment from "moment-timezone";

const googlemap = ref(null);

const currentLocationAddress = ref("");

const locationSearch = ref("");

const currentPlaceID = ref("");

const markerList = ref<Marker[]>([]);

const selectedMarker = ref<string[]>([]);

const timezoneLocalTime = ref("");

const headers = [{ title: "Address", key: "address", sortable: false }];

function deleteMarker() {
  markerList.value = markerList.value.filter(
    (m) => !selectedMarker.value.includes(m.placeId)
  );
  selectedMarker.value = [];
}

async function getPlaceAutoComplete(location: string) {
  //@ts-ignore
  const service = new google.maps.places.AutocompleteService();

  service.getPlacePredictions(
    {
      input: location,
    },
    function (predictions: any, status: any) {
      if (predictions.length == 0) {
        alert("No result found");
        return;
      }

      if (currentPlaceID.value == predictions[0].place_id) {
        alert("Same with previous search");
        return;
      }

      currentPlaceID.value = predictions[0].place_id;

      //@ts-ignore
      const geocoder = new google.maps.Geocoder();

      geocoder.geocode(
        { placeId: predictions[0].place_id },
        function (results: any, status: any) {
          if (status == "OK") {
            //@ts-ignore
            googlemap.value!.panTo?.({
              lat: results[0].geometry.location.lat(),
              lng: results[0].geometry.location.lng(),
            });

            if (
              markerList.value.filter(
                (m) => m.placeId == predictions[0].place_id
              ).length > 0
            ) {
              return;
            }

            markerList.value.push({
              latlng: {
                lat: results[0].geometry.location.lat(),
                lng: results[0].geometry.location.lng(),
              },
              address: results[0].formatted_address,
              placeId: predictions[0].place_id,
            });

            getTimezone(
              results[0].geometry.location.lat(),
              results[0].geometry.location.lng()
            );
          } else {
            alert(
              "Geocode was not successful for the following reason: " + status
            );
          }
        }
      );

      locationSearch.value = "";
    }
  );
}

async function getTimezone(lat: number, lng: number) {
  const res = await fetch(
    `https://maps.googleapis.com/maps/api/timezone/json?location=${lat},${lng}&timestamp=${
      new Date().getTime() / 1000
    }&key=${import.meta.env.VITE_GOOGLE_MAPS_API_KEY}`
  );
  const data = await res.json();

  console.log("timezone", data);

  const now = moment(); // Get the current time in UTC

  // Convert the current time to the local time based on the timeZoneId
  const localTime = now.tz(data.timeZoneId).format("YYYY-MM-DD HH:mm:ss");

  timezoneLocalTime.value = `${data.timeZoneName} ${localTime}`;

  console.log("Local Time:", localTime);
}

async function getLocationTextByLatLng(lat: number, lng: number) {
  const res = await fetch(
    `https://maps.googleapis.com/maps/api/geocode/json?latlng=${lat},${lng}&key=${
      import.meta.env.VITE_GOOGLE_MAPS_API_KEY
    }`
  );
  const data = await res.json();
  currentLocationAddress.value = data.results[0].formatted_address;
}

function getCurrentLocation() {
  if ("geolocation" in navigator) {
    navigator.geolocation.getCurrentPosition(
      function (position) {
        const latitude = position.coords.latitude;
        const longitude = position.coords.longitude;
        //@ts-ignore
        googlemap.value!.panTo?.({ lat: latitude, lng: longitude });
        getLocationTextByLatLng(latitude, longitude);
      },
      function (error) {
        alert("Error getting location:" + error.message);
      }
    );
  } else {
    alert("Geolocation is not available.");
  }
}

const center = { lat: 43.7482684, lng: -79.2920163 };
</script>
