<template>
  <div id="app">
    <div class="row no-gutters">
      <div class="col-sm-3">
        <div class="toolbox">
          <div class="sticky-top bg-white shadow-sm p-2">
            <div class="form-group d-flex">
              <label for="cityName" class="mr-2 col-form-label text-right">縣市</label>
              <div class="flex-fill">
                <select
                  id="cityName"
                  class="form-control"
                  v-model="select.city"
                  @change="removeMarker(); updateMap()"
                >
                  <option value>--請選擇縣市--</option>
                  <option :value="c.CityName" v-for="c in cityName" :key="c.id">{{c.CityName}}</option>
                </select>
              </div>
            </div>
            <div class="form-group d-flex">
              <label for="area" class="mr-2 col-form-label text-right">地區</label>
              <div class="flex-fill">
                <select
                  id="area"
                  class="form-control"
                  v-model="select.address"
                  @change="updataSelect"
                >
                  <option value>-- 請選擇區域--</option>
                  <option
                    :value="add.AreaName"
                    v-for="add in cityName.find((item) => item.CityName === select.city).AreaList"
                    :key="add.id"
                  >{{add.AreaName}}</option>
                </select>
              </div>
            </div>
            <p class="mb-0 small text-muted text-right">請先選擇區域查看（綠色表示還有口罩）</p>
          </div>
          <ul class="list-group">
            <template v-for="item in data">
              <a
                @click="moveTo(item)"
                class="list-group-item text-left"
                v-if="item.properties.town === select.address"
                :key="item.id"
              >
                <h3>藥局名稱:{{item.properties.name}}</h3>
                <p
                  class="mb-0"
                >成人口罩：{{item.properties.mask_adult}} | 兒童口罩：{{item.properties.mask_child}}</p>
                <p class="mb-0">
                  地址：
                  <a
                    :href="`https://www.google.com.tw/maps/place/${item.properties.address}`"
                    target="_blank"
                    title="Google Map"
                  >{{item.properties.address}}</a>
                </p>
              </a>
            </template>
          </ul>
        </div>
      </div>
      <div class="col-sm-9">
        <div id="map"></div>
      </div>
    </div>
  </div>
</template>

<script>
import L from 'leaflet'
import cityName from './assets/cityName.json'

let osmMap = []
const myIcon = {}
const icons = {
  blue: new L.Icon({
    iconUrl: 'https://unpkg.com/leaflet@1.6.0/dist/images/marker-icon-2x.png',
    ...myIcon
  }),
  grey: new L.Icon({
    iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-grey.png',
    ...myIcon
  })
}

console.log('icons', icons)
export default {
  name: 'App',
  data () {
    return {
      data: [],
      cityName,
      select: {
        city: '臺北市',
        address: '內湖區'
      }
    }
  },
  methods: {
    updateMap () {
      const vm = this
      const pharmacies = vm.data.filter(item => {
        // 選取市區時
        if (!vm.select.address) {
          return item.properties.county === vm.select.city
        }
        return item.properties.town === vm.select.address
      })
      pharmacies.forEach(item => {
        // 解構物件
        const { geometry, properties } = item
        L.marker([
          // 取得座標位置並新增
          geometry.coordinates[1],
          geometry.coordinates[0]
        ]).addTo(osmMap).bindPopup(`
        <strong>${properties.name}</strong><br>
        口罩剩:成人-${properties.mask_adult}/個
        兒童-${properties.mask_child}/個<br>
        地址:<a href="https://www.google.com.tw/maps/place/${properties.address}">${properties.address}</a><br>
        電話:${properties.phone}<br>
        最後更新時間:${properties.updated}<br>
        `)
      })
      // 載入時皆帶入第一個藥局位置
      vm.panTo(pharmacies[0])
    },
    removeMarker () {
      osmMap.eachLayer(layer => {
        if (layer instanceof L.Marker) {
          osmMap.removeLayer(layer)
        }
      })
    },
    panTo (item) {
      osmMap.panTo([
        item.geometry.coordinates[1],
        item.geometry.coordinates[0]
      ])
    },
    updataSelect () {
      const vm = this
      vm.removeMarker()
      vm.updateMap()
      // 選取地區的名稱一樣時回傳
      const pharmacies = vm.data.find(item => {
        return item.properties.town === vm.select.address
      })
      osmMap.panTo([
        pharmacies.geometry.coordinates[1],
        pharmacies.geometry.coordinates[0]
      ])
    },
    moveTo (item) {
      const { geometry, properties } = item
      const icon = properties.mask_adult ? icons.blue : icons.grey
      osmMap.panTo([
        geometry.coordinates[1],
        geometry.coordinates[0]
      ])
      L.marker([
        item.geometry.coordinates[1],
        item.geometry.coordinates[0]], {
        icon
      }).addTo(osmMap).bindPopup(`<strong>${properties.name}</strong> <br>
        口罩剩餘：<strong>成人 - ${properties.mask_adult ? `${properties.mask_adult} 個` : '未取得資料'}/ 兒童 - ${properties.mask_child ? `${properties.mask_child} 個` : '未取得資料'}</strong><br>
        地址: <a href="https://www.google.com.tw/maps/place/${properties.address}" target="_blank">${properties.address}</a><br>
        電話: ${properties.phone}<br>
        <small>最後更新時間: ${properties.updated}</small>`).openPopup()
    }
  },
  mounted () {
    const vm = this
    // 起始位置
    osmMap = L.map('map', {
      center: [25.03, 121.55],
      zoom: 15
    })

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution:
        '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
      maxZoom: 18
    }).addTo(osmMap)

    const url =
      'https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json'
    vm.$http.get(url).then(response => {
      vm.data = response.data.features
      vm.updateMap()
    })
  }
}
</script>

<style lang="scss">
@import "bootstrap/scss/bootstrap";

#map {
  height: 100vh;
}
.highlight {
  background: #e9ffe3;
}
.toolbox {
  height: 100vh;
  overflow-y: auto;
  a {
    cursor: pointer;
  }
}
</style>
