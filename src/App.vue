<template>
   <div class="row no-gutters" id="app">
      <div class="col-sm-4">
         <div class="toolbox">
            <div class=" p-2 searchBar">
               <h1 class="h3">
                  <img :src="icon.logo" class="logo" /><span>口罩即時查</span>
               </h1>
               <div class="form-group d-flex ">
                  <label for="cityName" class="mr-2 col-form-label text-right"
                     >縣市</label
                  >
                  <div class="flex-fill">
                     <select
                        id="cityName"
                        class="form-control"
                        v-model="select.city"
                        @change="updateSelect"
                     >
                        <option
                           :value="c.CityName"
                           :key="idx"
                           v-for="(c, idx) in cityData"
                           >{{ c.CityName }}</option
                        >
                     </select>
                  </div>
               </div>
               <div class="form-group d-flex">
                  <label for="area" class="mr-2 col-form-label text-right">地區</label>
                  <div class="flex-fill">
                     <select
                        id="area"
                        class="form-control"
                        v-model="select.area"
                        @change="updateSelect"
                     >
                        <option value="">-- Select One --</option>
                        <option
                           v-for="(a, idx) in cityData.find(
                              (i) => i.CityName === select.city
                           ).AreaList"
                           :key="idx"
                           :value="a.AreaName"
                           >{{ a.AreaName }}</option
                        >
                     </select>
                  </div>
               </div>
               <h2 class="h4 buyDay">今天是 <span>奇數</span> 購買日</h2>
               <p class="mb-0 small text-muted">
                  請先選擇區域查看（綠色表示還有口罩）
               </p>
            </div>
            <div class="dataList">
               <ul class="list-group ">
                  <template v-for="(item, key) in activeData">
                     <a
                        class=" text-left dataItem shadow-sm"
                        :key="key"
                        @click="panTo(item)"
                     >
                        <div class="d-flex maskTypeBox">
                           <div
                              class="maskType"
                              :class="[item.properties.mask_adult > 0 ? 'safe' : 'out']"
                           >
                              成人口罩數量
                              <br />
                              <span>{{ item.properties.mask_adult }}</span> 片
                              <img
                                 :src="
                                    item.properties.mask_adult > 0 ? icon.full : icon.none
                                 "
                              />
                           </div>
                           <div
                              class="maskType"
                              :class="[item.properties.mask_child > 0 ? 'safe' : 'out']"
                           >
                              兒童口罩數量
                              <br />
                              <span>{{ item.properties.mask_child }}</span> 片
                              <img
                                 :src="
                                    item.properties.mask_child > 0 ? icon.full : icon.none
                                 "
                              />
                           </div>
                        </div>
                        <h3 class="h5">{{ item.properties.name }}</h3>
                        <p class="mb-0 d-flex text">
                           <span> 地址：{{ item.properties.address }} </span>
                           <a
                              :href="
                                 `https://www.google.com.tw/maps/place/${item.properties.address}`
                              "
                              target="_blank"
                              title="Google Map"
                           >
                              於地圖查看
                           </a>
                        </p>
                        <p class="mb-0 d-flex text">
                           <span> 電話：{{ item.properties.phone }} </span>
                           <a
                              :href="`tel:${item.properties.phone}`"
                              target="_blank"
                              title="Google Map"
                           >
                              撥打電話</a
                           >
                        </p>
                     </a>
                  </template>
               </ul>
            </div>
         </div>
      </div>
      <div class="col-sm-8">
         <div id="map"></div>
      </div>
   </div>
</template>

<script>
import L from "leaflet";
import CityCountyData from "./assets/CityCountyData.json";
import MaskData from "./assets/MaskData.json";

const iconsConfig = {
   iconSize: [25, 41],
   iconAnchor: [12, 41],
   popupAnchor: [1, -34],
   shadowSize: [41, 41],
};

const icons = {
   green: new L.Icon({
      iconUrl:
         "https://cdn.rawgit.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png",
      ...iconsConfig,
   }),
   grey: new L.Icon({
      iconUrl:
         "https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-grey.png",
      ...iconsConfig,
   }),
};

const popupTemplate = (properties) => `<strong>${properties.name}</strong> <br>
    口罩剩餘：<strong>成人 - ${
       properties.mask_adult ? `${properties.mask_adult} 個` : "未取得資料"
    }/ 兒童 - ${
   properties.mask_child ? `${properties.mask_child} 個` : "未取得資料"
}</strong><br>
    地址: <a href="https://www.google.com.tw/maps/place/${
       properties.address
    }" target="_blank">${properties.address}</a><br>
    電話: ${properties.phone}<br>
    <small>最後更新時間: ${properties.updated}</small>`;

let osmMap = {};

const osm = {
   addMarker: (x, y, properties) => {
      const icon =
         properties.mask_adult || properties.mask_child ? icons.green : icons.grey;
      const marker = L.marker([x, y], { icon })
         .addTo(osmMap)
         .bindPopup(popupTemplate(properties));
      return marker;
   },
   removeMarker: () => {
      osmMap.eachLayer((layer) => {
         if (layer instanceof L.Marker) {
            osmMap.removeLayer(layer);
         }
      });
   },
   panTo: ([x, y], markers) => {
      osmMap.panTo([x, y]);
      const marker = markers.find(
         (item) => item.getLatLng().lat === x && item.getLatLng().lng === y
      );
      marker.openPopup();
   },
};

export default {
   name: "App",
   data: () => ({
      data: [],
      cityData: CityCountyData,
      markers: [],
      select: {
         city: "臺北市",
         area: "大安區",
      },
      icon: {
         full: require("./assets/ic_stock_full@2x.png"),
         none: require("./assets/ic_stock_none@2x.png"),
         logo: require("./assets/logo.png"),
      },
   }),
   mounted() {
      const url =
         "https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json";

      //取得口罩資料 (以防之後api無資料 先複製一分在本地端)
      // this.$http.get(url).then((res) => {
      //    this.data = res.data.features;
      // this.updateMarker();
      // });

      //預設台北市座標
      osmMap = L.map("map", { center: [25.03, 121.55], zoom: 15 });

      //取得地圖圖層
      L.tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
         attribution:
            '<a target="_blank" href="https://www.openstreetmap.org/">© OpenStreetMap 貢獻者</a>',
         maxZoom: 18,
      }).addTo(osmMap);

      //本地端舊資料
      this.data = MaskData;
      this.updateMarker();
   },
   methods: {
      updateMarker() {
         const activeArea = this.activeData;
         activeArea.forEach((e, idx) => {
            const { geometry, properties } = e;
            const [y, x] = geometry.coordinates;
            this.markers.push(osm.addMarker(x, y, properties));
         });
         if (activeArea.length === 0) return;
         this.panTo(activeArea[0]);
      },
      removeOldMarker() {
         this.markers = [];
         osm.removeMarker();
      },
      panTo(pharmacy) {
         const [y, x] = pharmacy.geometry.coordinates;
         osm.panTo([x, y], this.markers);
      },
      updateSelect(e) {
         if (e.target.id === "cityName") {
            this.select.area = "";
         }
         this.removeOldMarker();
         this.updateMarker();
      },
   },
   computed: {
      activeData: function() {
         return this.data.filter(
            (item) =>
               item.properties.county === this.select.city &&
               (this.select.area === "" || item.properties.town === this.select.area)
         );
      },
   },
};
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
   padding: 20px 30px;
   background-color: #eee;
   font-family: "微軟正黑體";
   color: #002c36;
   .h3 {
      * {
         vertical-align: middle;
      }
   }
   .logo {
      width: 43px;
      height: 43px;
   }
   a {
      cursor: pointer;
   }
   .dataItem {
      margin: 10px 3px;
      background-color: #fff;
      padding: 10px;
      border-radius: 8px;
      font-size: 13px;
      a {
         text-align: right;
         white-space: nowrap;
      }
      .text {
         justify-content: space-between;
      }
   }
   .maskTypeBox {
      margin-bottom: 5px;
   }
   .maskType {
      color: #fff;
      width: calc(50% - 4px);
      border-radius: 10px;
      padding: 10px;
      position: relative;
      span {
         font-weight: bold;
         font-size: 20px;
      }
      img {
         position: absolute;
         right: 0px;
         top: 10px;
         width: 50px;
         height: 50px;
      }
   }
   .maskType.safe {
      background-color: #25ab07;
   }
   .maskType.out {
      background-color: #aaa;
   }
   .maskType + .maskType {
      margin-left: 8px;
   }
}
.searchBar {
   height: 25%;
   .buyDay {
      span {
         font-size: 36px;
         font-weight: bold;
      }
   }
}
.dataList {
   overflow-y: auto;
   overflow-x: visible;
   height: 75%;
}
</style>
