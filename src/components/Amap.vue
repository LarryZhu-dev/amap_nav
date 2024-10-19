<script setup lang="ts">
import AMapLoader from '@amap/amap-jsapi-loader';
import { ref, onMounted, h } from 'vue'
import { ElButton, ElIcon, ElAutocomplete } from 'element-plus'
import { LocationFilled } from '@element-plus/icons-vue'
import { pointToLineDistance, nearestPointOnLine, distance as pointToPointDistance, along, bearing } from '@turf/turf'
import car from '../assets/imgs/car.png'


const actionIconDict: { [key: string]: string } = {
  '左转': 'icon-xiangzuozhuan',
  '右转': 'icon-xiangyouzhuan',
  '直行': 'icon-xiangshang',
  '向左前方行驶': 'icon-xiangzuozhuan',
  "向右前方行驶": "icon-xiangyouzhuan",
  "向左后方行驶": "icon-xiangzuozhuan",
  "向右后方行驶": "icon-xiangyouzhuan",
  "左转调头": "icon-xiangzuozhuan",
  "靠左": "icon-xiangzuozhuan",
  "靠右": "icon-xiangyouzhuan",
  "进入环岛": "icon-xiangshang",
  "离开环岛": "icon-xiangshang",
  "减速行驶": "icon-xiangshang"
}

let AMap: any
let map: any
let driving: any
let location: any
const points = ref<{
  keyword: string,
  lnglat: number[],
}[]>([
  { keyword: '北京物资学院', lnglat: [116.638339, 39.930726] }, //起始点坐标
  { keyword: '天安门', lnglat: [116.397455, 39.909187] } //终点坐标
])
const currentRoute = ref<{
  distance?: number,
  duration?: number,
  policy?: string,
  steps?: {
    instruction: string,
    distance: string,
    action: string,
    icon: string,
    startPoint: number[],
    endPoint: number[],
    time: number,
    index: number
  }[]
}>({})
const currStep = ref<{
  instruction: string,
  distance: string,
  action: string,
  icon: string,
  startPoint: number[],
  endPoint: number[],
  time: number,
  index: number
}>()
const loading = ref(false)
const navStarted = ref(false)
const startTimestamp = ref(0)
const endTimestamp = ref(0)
const heading = ref(0)
const currentLocation = ref<number[]>([])


onMounted(async () => {
  // @ts-ignore
  window.movingDraw = true;
  await initAMap()
})
async function initAMap() {
  window._AMapSecurityConfig = {
    securityJsCode: import.meta.env.VITE_AMAP_SECURITY_CODE,
  };
  AMap = await AMapLoader.load({
    key: import.meta.env.VITE_AMAP_KEY, // 申请好的Web端开发者Key，首次调用 load 时必填
    version: "2.0", // 指定要加载的 JSAPI 的版本，缺省时默认为 1.4.15
    plugins: ["AMap.GeoJSON", "AMap.PlaceSearch", "AMap.Driving", "AMap.Geolocation", "AMap.Marker"], //需要使用的的插件列表，如比例尺'AMap.Scale'，支持添加多个如：['...','...']
    Loca: {                // 是否加载 Loca， 缺省不加载
      "version": '2.0.0'  // Loca 版本，缺省 1.3.2
    },
  })
  map = new AMap.Map("Amap", {
    center: [116.397428, 39.90923],
    zoom: 12,
    pitchEnable: true, // 是否允许设置俯仰角度
    viewMode: '3D',
    pitch: 50,
  });
  driving = new AMap.Driving({
    map: map,
  });
  location = new AMap.Geolocation({
    enableHighAccuracy: true,
    timeout: 10000,
    showButton: true,
    buttonPosition: 'RB',
    zoomToAccuracy: true,
    buttonOffset: new AMap.Pixel(10, 20),
  });
  getRoute()
}

function getRoute() {
  if (!points.value[0].lnglat.length || !points.value[1].lnglat.length) {
    return
  }
  loading.value = true
  driving.search(points.value[0].lnglat, points.value[1].lnglat, function (status: any, result: any) {
    if (status === 'complete') {
      console.log(result.routes[0])
      currentRoute.value = {
        distance: result.routes[0].distance,
        duration: result.routes[0].time,
        policy: result.routes[0].policy,
        steps: result.routes[0].steps.map((item: any, index: number) => {
          return {
            instruction: item.instruction,
            distance: meters2kilometers(item.distance),
            action: item.action,
            icon: actionIconDict[item.action],
            startPoint: [item.start_location.lng, item.start_location.lat],
            endPoint: [item.end_location.lng, item.end_location.lat],
            time: item.time,
            index: index
          }
        })
      }
    } else {
      console.log('获取驾车数据失败：' + result)
      currentRoute.value = {}
    }
    loading.value = false
  });
}

function meters2kilometers(meters: number) {
  return meters > 1000 ? (meters / 1000).toFixed(2) + '公里' : meters + '米'
}
function seconds2minOrHour(seconds: number) {
  return seconds > 3600 ? (seconds / 3600).toFixed(2) + '小时' : (seconds / 60).toFixed(0) + '分钟'
}

function handleSelectStart(item: any) {
  points.value[0].keyword = item.raw.name
  points.value[0].lnglat = [item.raw.location.lng, item.raw.location.lat]
  getRoute()
}
function handleSelectEnd(item: any) {
  points.value[1].keyword = item.raw.name
  points.value[1].lnglat = [item.raw.location.lng, item.raw.location.lat]
  getRoute()
}

function querySearchAsync(queryString: string, cb: (arg: any) => void) {
  const placeSearch = new AMap.PlaceSearch({
    city: '北京',
    pageSize: 5,
    pageIndex: 1,
    citylimit: true,
    extensions: 'all',
  });
  placeSearch.search(queryString, function (status: any, result: any) {
    if (status === 'complete') {
      const res = result.poiList.pois.map((item: any) => {
        return {
          value: item.name,
          label: item.name,
          raw: item
        }
      })
      cb(res)
    }
  });
}
function startNav() {
  if (navStarted.value) {
    return
  }
  startTimestamp.value = new Date().getTime()
  navStarted.value = true

  // 创建一个 icon
  var endIcon = new AMap.Icon({
    size: new AMap.Size(40, 40),
    image: car,
    imageSize: new AMap.Size(40, 40),
  });

  // 将 icon 传入 marker
  var endMarker = new AMap.Marker({
    position: new AMap.LngLat(0, 0),
    icon: endIcon,
    offset: new AMap.Pixel(-13, -30)
  });

  // 将 markers 添加到地图
  map.add([endMarker]);
  getCurrentLocation((currLocation: number[]) => {
    endMarker.setPosition(currLocation)
    routeNav(currLocation)
  })
}

function routeNav(currLocation: number[]) {
  if (checkIsCloseToEndpoint()) {
    console.log('导航结束')
    navStarted.value = false
    currStep.value = undefined
    return
  }
  currStep.value = getCurrStep(currLocation)
  heading.value = -bearing(currStep.value.startPoint, currStep.value.endPoint)
  map.setCenter(currLocation)
  map.setZoom(16)
  map.setRotation(heading.value)
}

function checkIsCloseToEndpoint() {
  if (!currStep.value?.endPoint) {
    return false
  }
  const distance = pointToPointDistance({
    type: 'Point',
    coordinates: currStep.value!.endPoint
  }, currentRoute.value.steps![currentRoute.value.steps!.length - 1].endPoint, { units: 'meters' })
  return distance < 10
}

function getCurrStep(currLocation: number[]) {
  const paths = currentRoute.value.steps!.map((item) => {
    return item.startPoint
  })
  const nearestPoint = nearestPointOnLine({
    type: 'Feature',
    properties: {},
    geometry: {
      type: 'LineString',
      coordinates: paths
    }
  }, currLocation)
  return currentRoute.value.steps![nearestPoint.properties.index]
}

const mock = true
const mockSpeed = 1000
let mockTimer: any
function getCurrentLocation(cb: (arg: number[]) => void) {
  if (mock) {
    clearInterval(mockTimer)
    mockTimer = setInterval(() => {
      let currentTimestamp = new Date().getTime()
      let runTime = (currentTimestamp - startTimestamp.value) / 1000
      for (let item of currentRoute.value.steps!) {
        if (runTime < item.time) {
          let distance = pointToPointDistance(item.startPoint, item.endPoint)
          let speed = (distance / item.time) * mockSpeed
          let currentDistance = speed * runTime
          try {
            let currentLocation = along({
              type: 'LineString',
              coordinates: [item.startPoint, item.endPoint]
            }, currentDistance, { units: 'meters' })
            let position = nearestPointOnLine({
              type: 'Feature',
              properties: {},
              geometry: {
                type: 'LineString',
                coordinates: [item.startPoint, item.endPoint]
              }
            }, currentLocation.geometry.coordinates)
            cb(position.geometry.coordinates)
          } catch (e) {
            console.log('e::: ', e);
            continue
          }
          break
        }
      }
    }, 100)
    return
  }
  location.getCurrentPosition(function (status: string, result: any) {
    if (status == 'complete') {
      console.log('result::: ', result);
    } else {
      console.log('result::: ', result);
    }
  });
}
</script>

<template>
  <div id="Amap"></div>
  <div class="optionsMenu">
    <div class="optionsBody">
      <div class="searchBox" v-if="!currStep">
        <div class="inputs">
          <ElAutocomplete placeholder="请输入起点" v-model="points[0].keyword" size="large" @select="handleSelectStart"
            :fetch-suggestions="querySearchAsync">
            <template #prefix>
              <ElIcon color="#00b144" size="24">
                <LocationFilled />
              </ElIcon>
            </template>
          </ElAutocomplete>
          <ElAutocomplete placeholder="请输入终点" v-model="points[1].keyword" size="large" @select="handleSelectEnd"
            :fetch-suggestions="querySearchAsync">
            <template #prefix>
              <ElIcon color="#d32f19" size="24">
                <LocationFilled />
              </ElIcon>
            </template>
          </ElAutocomplete>
        </div>
      </div>
      <div class="currStep" v-else>
        <div class="stepInfo">
          <span class="instruction">
            <span class="iconfont" :class="currStep.icon"></span>
            <span>{{ currStep.instruction }}</span>
          </span>
        </div>
      </div>
      <div class="starts" v-if="currentRoute.policy">
        <div class="routeInfo" v-if="!currStep">
          <span class="policy">
            <span>已找到路线：</span>
            <span> {{ currentRoute.policy }}</span>
          </span>
          <div class="otherInfo">
            <span class="distance">
              <span class="iconfont icon-juli"></span>
              <span>距离：</span>
              <span>{{ meters2kilometers(currentRoute.distance!) }}</span>
            </span>
            <span class="duration">
              <span class="iconfont icon-shijian"></span>
              <span>预计行驶时间：</span>
              <span>{{ seconds2minOrHour(currentRoute.duration!) }}</span>
            </span>
          </div>
        </div>
        <ElButton type="primary" size="large" :loading="loading" @click="startNav">{{ currStep ? '导航中...' : '开始导航' }}
        </ElButton>
      </div>
    </div>
  </div>
</template>

<style scoped lang="less">
.read-the-docs {
  color: #888;
}

#Amap {
  width: 100%;
  height: 100%;
}

.optionsMenu {
  position: fixed;
  display: flex;
  flex-direction: column;
  align-items: center;
  top: 0;
  left: 0;
  background: #fff;
  border-radius: 6px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
  color: #666;
  padding: 26px;
  width: 100%;

  .logo {
    width: 100%;
    display: flex;
    align-items: center;
    justify-content: flex-start;
    padding: 12px;

    img {
      width: 20%;
    }
  }

  .optionsBody {
    flex: 1;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    gap: 18px;
    width: 100%;
    height: 100%;

    .starts {
      position: fixed;
      bottom: 0;
      left: 0;
      width: 100%;
      padding: 26px 30px;
      background: #fff;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;

      .routeInfo {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 10px;
        padding: 12px 0;

        .policy {
          font-size: 24px;
          color: #333;

          span:first-child {
            color: #999;
            font-size: 16px;
          }
        }

        .otherInfo {
          display: flex;
          gap: 18px;

          .distance,
          .duration {
            display: flex;
            align-items: center;
            gap: 3px;
            font-size: 14px;
            color: #999;

            span:last-child {
              font-size: 16px;
              font-weight: bold;
              color: #409eff;
            }

            .iconfont {
              font-size: 16px;
              color: #409eff;
            }
          }
        }
      }

      button {
        width: 100%;
      }
    }

    .currStep {
      width: 100%;
      background: #fff;
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: center;

      .stepInfo {
        display: flex;
        flex-direction: column;
        align-items: center;
        gap: 10px;
        padding: 12px 0;

        .instruction {
          font-size: 18px;
          color: #333;
          display: flex;
          align-items: center;
          gap: 24px;

          .iconfont {
            font-size: 24px;
            color: #409eff;
          }
        }
      }
    }

    .searchBox {
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: flex-end;
      gap: 10px;
      width: 100%;
      background: #f5f5f5;
      padding: 16px;
      border-radius: 6px;

      .el-input {
        width: 100%;
        font-size: 16px;
      }

      .inputs {
        display: flex;
        flex-direction: column;
        width: 100%;
        gap: 10px;

        .el-input {
          width: 100%;
        }
      }
    }

    .submit {
      display: flex;
      justify-content: center;
      bottom: 20px;
      background: #fff;
    }
  }
}
</style>
<style>
.amap-content-body {
  color: #000;
}
</style>