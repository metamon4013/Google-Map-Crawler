<template>
  <div>
    <h1>商業店舗検索</h1>
    <p>
      <select v-model="selectedType" @change="show">
        <option value="all">すべて</option>
        <option v-for="storeType in Object.keys(types)" :key="storeType" :value="storeType">{{ types[storeType] }}</option>
      </select>
    </p>
    <p style="font-size: 20px;">
      このお店は<span style="color: red;">{{ storeName }}</span>です！
    </p>
    <div id="map" style="height: 800px;"></div>
  </div>
</template>

<script setup lang="ts">
import { Loader } from '@googlemaps/js-api-loader';
const runTimeConfig = useRuntimeConfig();

const loader = new Loader({
  apiKey: runTimeConfig.public.api_key,
  version: "weekly",
  libraries: ["places"]
});

const selectedType = ref('all')
const show = () => {
  console.log(selectedType.value)
}

const Dinos: google.maps.LatLngLiteral = { lat: 42.34250610228756, lng: 141.02638446632784 }

const storeName = ref('')
const types: { [key: string]: string } = {
  "amusement_park": "アミューズメントパーク",
  // "restaurant": "レストラン",
  // "cafe": "カフェ",
  // "shopping_mall": "ショッピングモール"
}

onMounted(() => {
  loader
    .importLibrary('maps')
    .then(({Map}) => {
      const element = document.getElementById("map") as HTMLElement
      const map = new Map(element, {
        zoom: 10,
        center: Dinos,
        mapId: 'DEMO_MAP_ID',
      });
      // ディノスパークの位置を表示
      loader
        .importLibrary('marker')
        .then(({AdvancedMarkerElement, PinElement}) => {
          const pinView = new PinElement({
            background: '#FBBC04',
          })
          new AdvancedMarkerElement({
            map: map,
            position: Dinos,
            title: 'ディノスパーク室蘭中央',
            content: pinView.element
          })
        })
        .catch((e) => {
          console.log(e)
        });

      // 商業施設のマーカーを配置
      loader
        .importLibrary('places')
        .then(({PlacesService}) => {
          const placeService = new PlacesService(map)
          Object.keys(types).forEach(type => {
            placeService.nearbySearch({
              type,
              // rankBy: RankBy.DISTANCE,
              location: Dinos,
              radius: 50 * 1000
            }, (results, status, pagination) => {
              if (pagination?.hasNextPage) {
                pagination.nextPage()
              }
              
              console.log(results?.length)
              results?.forEach(store => {
                setMarker(store)
              })
            })
          })
        })

      const selectedRoute: Ref<google.maps.DirectionsRenderer | null> = ref(null)
      const selectedWindow: Ref<google.maps.InfoWindow | null> = ref(null)
      
      const setMarker = (storeInfo: google.maps.places.PlaceResult) => {
        if (storeInfo.url) {
          console.log(storeInfo)
        }
        loader
          .importLibrary('marker')
          .then(({Marker}) => {
            const storeLatLng = storeInfo.geometry?.location
            if (storeLatLng) {
              const store = new Marker({
                map: map,
                position: storeLatLng,
                title: storeInfo.name
              })
              store.addListener("click", () => {
                loader
                  .importLibrary('routes')
                  .then(({DirectionsRenderer}) => {
                    const renderRoute = new DirectionsRenderer({map})
                    renderRoute.setMap(null)
                  })
                // 距離を計算する
                const directDistance = Math.round(6378.137*Math.acos(Math.sin(storeLatLng.lat()/180*Math.PI)*Math.sin(Dinos.lat/180*Math.PI)+Math.cos(storeLatLng.lat()/180*Math.PI)*Math.cos(Dinos.lat/180*Math.PI)*Math.cos((Dinos.lng-storeLatLng.lng())/180*Math.PI)))
                // 道のりを計算する
                loader
                  .importLibrary('routes')
                  .then(({DistanceMatrixService, TravelMode}) => {
                    const distanceService = new DistanceMatrixService()
                    distanceService.getDistanceMatrix({
                      destinations: [storeLatLng],
                      origins: [Dinos],
                      travelMode: TravelMode.DRIVING
                    }, (road) => {
                      const roadDistance = Number(road?.rows[0].elements[0].distance.text.split(" ")[0])
                      loader
                        .importLibrary('places')
                        .then(({PlacesService}) => {
                          const placeSearch = new PlacesService(map)
                          console.log(storeInfo.place_id)
                          placeSearch.getDetails({placeId: storeInfo.place_id}, (result) => {
                            console.log(result?.website)
                          })

                        })
                      // ポップアップを表示する
                      loader
                        .importLibrary('maps')
                        .then(({ InfoWindow }) => {
                          const infoWindow = new InfoWindow({
                            content: `<h3>${storeInfo.name}</h3>
                            <p>直線距離は <b>${directDistance}km</b> です。</p>
                            <p>道のり距離は <b>${roadDistance}km</b> です。</p>
                            <p>住所は <b>${storeInfo.vicinity}</b> です。</p>
                            <p>ここは <b>${storeInfo.types?.length ? types[storeInfo.types[0]] : "業態不明"}</b> です。</p>
                            ${storeInfo.photos?.length
                              ? `<div width="250" style="display: flex; justify-content: center;">
                                  <img src="${storeInfo.photos[0].getUrl()}" height="100" />
                                </div>`
                              : ""
                            }
                            <p>${storeInfo.url}</p>
                            `,
                            ariaLabel: "中華料理 福楽餃子坊 新生町店"
                          })
                          console.log(storeInfo)
                          if (selectedWindow.value) {
                            selectedWindow.value.close()
                          }
                          infoWindow.open({
                            anchor: store,
                            map
                          })
                          selectedWindow.value = infoWindow
                        })
                    })
                  })
                // 道のりを表示する
                loader
                  .importLibrary('routes')
                  .then(({DirectionsService, DirectionsRenderer, TravelMode}) => {
                    const directionService = new DirectionsService()
                    directionService.route({
                      destination: storeLatLng,
                      origin: Dinos,
                      travelMode: TravelMode.DRIVING
                    }, (direction) => {
                      const renderer = new DirectionsRenderer({
                        map,
                        directions: direction
                      })
                      if (selectedRoute.value) {
                        selectedRoute.value.setMap(null)
                      }
                      renderer.setMap(map)
                      selectedRoute.value = renderer
                    })
                  })
                storeName.value = storeInfo.name || ""
              })
            }
          })
          .catch((e) => {
            console.log(e)
          });
      }
    })
    .catch((e) => {
      console.log(e)
    });
    
    
})
</script>
