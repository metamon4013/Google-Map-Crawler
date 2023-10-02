<template>
  <div>
    <div id="map" style="height: 800px;"></div>
  </div>
</template>

<script setup lang="ts">
import { Loader } from '@googlemaps/js-api-loader';
import { Title } from 'nuxt/dist/head/runtime/components';
import tokyo from '~/assets/tokyo.json'
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
      const Tokyo: google.maps.LatLngLiteral = {lat: 35.681, lng: 139.767}
      const map = new Map(element, {
        zoom: 10,
        center: Tokyo,
        mapId: 'DEMO_MAP_ID',
      });

      loader
        .importLibrary('places')
        .then(({PlacesService}) => {
          const placesSearvice = new PlacesService(map)
          tokyo.tokyo_cities.forEach((city) => {
            placesSearvice.textSearch({
              query: `${city}にあるホットヨガスタジオLAVA`
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
                    marker.setMap
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
