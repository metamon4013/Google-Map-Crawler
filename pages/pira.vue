<template>
  <div>
    <div id="map" style="height: 800px;"></div>
  </div>
</template>

<script setup lang="ts">
import { Loader } from '@googlemaps/js-api-loader';
import nihon from '~/assets/tokyo.json'
import america from '~/assets/america.json'
const runTimeConfig = useRuntimeConfig();

const loader = new Loader({
  apiKey: runTimeConfig.public.api_key,
  version: "weekly",
  libraries: ["places"]
});

onMounted(() => {
  loader
    .importLibrary('maps')
    .then(({Map}) => {
      const element = document.getElementById("map") as HTMLElement
      const Center: google.maps.LatLngLiteral = {lat: 38.898, lng: -77.0365}
      const map = new Map(element, {
        zoom: 10,
        center: Center,
        mapId: 'DEMO_MAP_ID',
      });

      loader
        .importLibrary('places')
        .then(({PlacesService}) => {
          const placesSearvice = new PlacesService(map)
          america.forEach((city) => {
            placesSearvice.textSearch({
              query: `${city}にあるPure Barre`
            }, (result, serviceStatus, pagination) => {
              console.log(pagination?.hasNextPage)
              console.log(result)
              loader
                .importLibrary('marker')
                .then(({Marker}) => {
                  result?.forEach(store => {
                    const marker = new Marker({
                      map,
                      position: store.geometry?.location,
                      title: store.name
                    })
                    if (store.name?.includes("Pure")) {
                      marker.setMap(map)
                    }
                  })
                })
              if (pagination?.hasNextPage) {
                pagination.nextPage()
              }
            })
          })
          
        })

    })
})
</script>
