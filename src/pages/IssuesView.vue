<template>
  <!-- eslint-disable -->
  <layout :source="title">
    <template slot="search-input">
      <div class="row search-bar-container">
        <div class="col-xs-12 col-sm-12 col-md-12 col-lg-4 p-0 text-right search-container">
          <img class="search-icon mt-0" :src="require('../assets/ic_search.svg')">
          <input type="text" class="beacon-search" v-model="search" placeholder="Search issue">
        </div>
        <div class="col-xs-12 col-sm-12 col-md-5 col-lg-3 p-0 text-right search-container">
          <search-group-filter ref="searchGroupFilter" v-model="groupFilter" />
        </div>
        <div class="col-xs-12 col-sm-12 col-md-7 col-lg-5 p-0 search-container">
          <button type="button" class="btn btn-reset ml-2" @click="resetFilter">Reset</button>
          <button type="button" class="btn btn-reset ml-2" @click="reload">Reload</button>
        </div>
      </div>
    </template>
    <template slot="body">
      <div class="row flex-grow-1" style="overflow: hidden">
        <div class="position-absolute w-100">
          <div id="view-switch" class="col-xs-12 col-sm-6 col-md-3 col-lg-2 btn-group mt-4" role="group" aria-label="Switch view">
            <button type="button" :class="'btn btn-view-switch ' + (viewMode === LIST ? 'btn-view-switch-active' : '')" @click="changeMode(LIST)"><img src="../assets/ic_list.svg"/></button>
            <button type="button" :class="'btn btn-view-switch ' + (viewMode === MAP ? 'btn-view-switch-active' : '')" @click="changeMode(MAP)"><img src="../assets/ic_map.svg"/></button>
          </div>
          <div id="view-switch" class="col-xs-12 col-sm-6 col-md-3 col-lg-2 btn-group mt-4" role="group" aria-label="Switch view">
            <button type="button" :class="'btn btn-view-switch ' + (statusFilter === ISSUES_FILTER_OPEN ? 'btn-view-switch-active' : '')" @click="changeIssuesFilter(ISSUES_FILTER_OPEN)">Open</button>
            <button type="button" :class="'btn btn-view-switch ' + (statusFilter === ISSUES_FILTER_CLOSED ? 'btn-view-switch-active' : '')" @click="changeIssuesFilter(ISSUES_FILTER_CLOSED)">Closed</button>
            <button type="button" :class="'btn btn-view-switch ' + (statusFilter === ISSUES_FILTER_ALL ? 'btn-view-switch-active' : '')" @click="changeIssuesFilter(ISSUES_FILTER_ALL)">All</button>
          </div>
        </div>
        <div class="container issues-table-container p-0" v-show="loaded && viewMode === LIST">
          <div class="row beacon-display m-4 p-4">
            <div class="col-12 p-0" v-show="tableData.length > 0">
              <simple-table responsive @change="reloadTableData" :cols="tableCols" :data="tableData" :meta="tableMeta" @rowClicked="showDetail"/>
            </div>
            <div class="col-12 p-0 text-center" v-show="tableData.length <= 0">
              <span class="text-muted">No issues found...</span>
            </div>
          </div>
        </div>
        <div id="map" class="issue-map" :style="{visibility: loaded && viewMode === MAP ? 'visible' : 'hidden'}">
        </div>
        <loader :visible="!loaded" :label="'Loading issues...'"/>
      </div>
    </template>
  </layout>
</template>

<script>
  import Layout from '../components/Layout'
  import SimpleTable from '../components/SimpleTable'
  import { mapActions, mapGetters } from 'vuex'
  import { LIST, MAP } from '../store/issues'
  import Loader from '../components/Loader'
  import { initMap } from '../service/googlemaps'
  import router from '../router/index'
  import merge from 'lodash/merge'
  import SearchGroupFilter from "../components/SearchGroupFilter";

  export default {
    components: {
      SearchGroupFilter,
      Layout,
      SimpleTable,
      Loader
    },
    name: 'Issues',
    data() {
      return {
        LIST: LIST,
        MAP: MAP,
        ISSUES_FILTER_OPEN: 'ISSUES_FILTER_OPEN',
        ISSUES_FILTER_CLOSED: 'ISSUES_FILTER_CLOSED',
        ISSUES_FILTER_ALL: 'ISSUES_FILTER_ALL',
        title: 'Issues',
        tableCols: [
          {
            title: 'Id',
            key: 'beacon.id'
          },
          {
            title: 'Name',
            key: 'beacon.name'
          },
          {
            title: 'Problem',
            key: 'problem'
          },
          {
            title: 'Description',
            key: 'problemDescription'
          },
          {
            title: 'Date',
            key: 'reportDate',
            type: 'date'
          },
          {
            title: 'Group',
            key: 'beacon.group.name'
          },
          {
            title: 'Battery',
            key: 'beacon.batteryLevel',
            type: 'battery-level'
          },
          {
            title: 'Status',
            key: 'beacon.status',
            type: 'beacon-status'
          },
          {
            title: 'Last modified',
            key: 'lastModified',
            type: 'date'
          },
          {
            title: '',
            key: 'resolved',
            type: 'issue-status'
          }
        ],
        mapData: [],
        tableData: [],
        tableMeta: {
          sorting: {
            col: 'lastModified',
            order: 'desc'
          },
          pagination: {
            offset: 0,
            page: 1,
            records: 10,
            recordsNumberList: [2, 5, 10, 20]
          }
        },
        search: '',
        groupFilter: '',
        loaded: false,
        map: null,
        mapBeacons: [],
        markers: [],
        beaconIds: [],
        addedOnMap: false,
        myPosition: null,
        clusterer: null,
        google: {},
        timers: [],
        statusFilter: null
      }
    },
    computed: {
      ...mapGetters('issues', [
        'issues',
        'viewMode'
      ])
    },
    watch: {
      search() {

        sessionStorage.setItem('issues_search', this.search)

        this.reloadTableData()
        this.$set(this, 'mapBeacons', this.mapData.slice(0))
      },
      issues() {
        this.reloadTableData()
        this.$set(this, 'loaded', true)
        this.$set(this, 'mapBeacons', this.mapData.slice(0)) // load map markers
      },
      groupFilter() {
        sessionStorage.setItem("group_filter", this.groupFilter)

        this.reloadTableData()
        this.$set(this, 'mapBeacons', this.mapData.slice(0))
      },
      mapBeacons() {
        // let newMarkers = []

        /*
        if (this.mapBeacons === null) {
          this.updateMarkers(newMarkers)
          return
        }
        */

        // if leaflet is not ready, skip
        if (!this.L)
          return

        // cancel previous marker timers
        while (this.timers.length > 0)
          clearTimeout(this.timers.shift());

        if (this.cluster == null)
        {
          this.cluster = this.L.markerClusterGroup();
          this.map.addLayer(this.cluster);
        }
        else
        {
          this.cluster.removeLayers(this.cluster.getLayers())
        }

        this.mapBeacons.forEach((beacon) => {
          let position = this.getPosition(beacon)

          if (position.lat !== 0 || position.lng !== 0) {

            var customIcon = this.L.icon({
              iconUrl: this.iconSvg(beacon),
              iconSize:     [24, 24], // size of the icon
              iconAnchor:   [12, 12], // point of the icon which will correspond to marker's location
            });

            let marker = this.L.marker([position.lat, position.lng], {icon: customIcon}) //.addTo(this.map);
            var popupContent = '<div class="popup-item"><h2 class="beacon-title">' + beacon.name + '</h2>' +
              '<h5 class="beacon-title">Id: ' + beacon.id + '</h5>' +
              '<span class="text-muted">Group: </span><span>' + beacon.group.name + '</span><br />' +
              '<span class="text-muted">POI name: </span><span>' + beacon.namePoi + '</span><br />' +
              '<div class="d-flex">' +
              '  <span><span class="text-muted">Battery:</span> ' + (beacon.batteryLevel != null && beacon.batteryLevel != undefined? beacon.batteryLevel: '?') + ' %</span><br />' +
              '  <div class="battery ml-2 ' + (beacon.batteryLevel == null || beacon.batteryLevel == undefined? 'unknown': (beacon.batteryLevel < 5? 'warning': '')) + '">' +
              '    <div class="chargestatus" style="' + (beacon.batteryLevel != null && beacon.batteryLevel != undefined? 'top:' + (100 - beacon.batteryLevel) + '%;height:' + beacon.batteryLevel + '%': '') + '"></div>' +
              '  </div>' +
              '</div>' +
              '<button id="beaconPopupLink" type="button" class="btn btn-popup-open mt-2">Open</button>'
            // marker.on('click', () => {
            //   router.push({name: 'beacon-detail', params: {id: beacon.id}})
            // })
            marker.bindPopup(popupContent).on("popupopen", () => {
              document.getElementById('beaconPopupLink').onclick = (e) => {
                e.preventDefault();
                router.push({name: 'issue-detail', params: {id: beacon.id}})
              }
            })
            let ccc = this.cluster
            // add marker async
            this.timers.push(setTimeout(function() { ccc.addLayer(marker); }, 200));

          }
        })

        // this.updateMarkers(newMarkers)
      }
    },
    methods: {
      ...mapActions('issues', [
        'fetchIssues',
        'clear'
      ]),
      ...mapActions('groups', [
        'fetchGroups',
        'clear'
      ]),
      getPosition(beacon) {
        if (beacon.lat !== 0 && beacon.lng !== 0) {
          return {
            lat: beacon.lat,
            lng: beacon.lng
          }
        } else if (beacon.info_lat !== 0 && beacon.info_lng !== 0) {
          return {
            lat: beacon.info_lat,
            lng: beacon.info_lng
          }
        }

        return {
          lat: 0,
          lng: 0
        }
      },
      showDetail(issue) {
        router.push({name: 'issue-detail-issue', params: {id: issue.beacon.id, issueId: issue.id}})
      },
      showMyPosition(success, failure) {
        let myPositionButtonIcon = document.getElementById('myLocationButtonIcon')
        let on = false
        let myLocationButtonBlinker = setInterval(function(){
          myPositionButtonIcon.src = on ? require('../assets/img/map/my_location.svg') : require('../assets/img/map/my_location_empty.svg')
          on = !on
        }, 500);
        navigator.geolocation.getCurrentPosition(position => {
          clearInterval(myLocationButtonBlinker)
          myPositionButtonIcon.src = require('../assets/img/map/my_location.svg')
          if (this.myPosition != null) {
            this.myPosition.setMap(null)
          }
          this.myPosition = new this.google.maps.Marker({
            position: {
              lat: position.coords.latitude,
              lng: position.coords.longitude
            },
            draggable: this.editing,
            map: this.map,
            icon: {
              url: require('../assets/img/map/map_icon_my_location.svg'),
              size: new this.google.maps.Size(24, 24),
              scaledSize: new this.google.maps.Size(24, 24),
              anchor: new this.google.maps.Point(12, 12)
            }
          })
          success(position)
        }, error => {
          clearInterval(myLocationButtonBlinker)
          myPositionButtonIcon.src = require('../assets/img/map/my_location.svg')
          failure(error)
        })
      },
      goToMyPosition() {
        this.showMyPosition(position => {
          this.map.panTo(new this.google.maps.LatLng(position.coords.latitude, position.coords.longitude))
          this.map.setZoom(16)
        }, () => {
          window.console.log("Please make sure you have allowed the acces to your location.")
        })
      },
      MyLocationControl(controlDiv) {
        this.controlUI = document.createElement('div')
        this.controlUI.style.backgroundColor = '#fff'
        this.controlUI.style.border = '2px solid #fff'
        this.controlUI.style.borderRadius = '3px'
        this.controlUI.style.boxShadow = '0 2px 6px rgba(0,0,0,.3)'
        this.controlUI.style.cursor = 'pointer'
        this.controlUI.style.marginBottom = '0px'
        this.controlUI.style.marginRight = '10px'
        this.controlUI.style.width = '40px'
        this.controlUI.style.height = '40px'
        this.controlUI.style.textAlign = 'center'
        this.controlUI.title = 'Click to recenter the map'
        controlDiv.appendChild(this.controlUI)

        this.controlText = document.createElement('div')
        this.controlText.style.color = 'rgb(25,25,25)'
        this.controlText.style.fontFamily = 'Roboto,Arial,sans-serif'
        this.controlText.style.fontSize = '16px'
        this.controlText.style.lineHeight = '38px'
        this.controlText.style.paddingLeft = '5px'
        this.controlText.style.paddingRight = '5px'
        this.controlText.innerHTML = '<img id="myLocationButtonIcon" src="' + require('../assets/img/map/my_location.svg') +  '" class="map-control"/>'
        this.controlUI.appendChild(this.controlText);

        this.addClickListener = function(callback) {
          this.controlUI.addEventListener('click', callback)
        }

      },
      changeMode(mode) {
        this.$store.dispatch('issues/setViewMode', mode)
      },
      changeIssuesFilter(statusFilter) {
        this.statusFilter = statusFilter
        sessionStorage.setItem("issues_status_filter", this.statusFilter)

        this.reloadTableData()
        this.$set(this, 'mapBeacons', this.mapData.slice(0))
      },
      reloadTableData(params = {}) {
        params = merge({
          pagination: this.tableMeta.pagination,
          sorting: this.tableMeta.sorting,
          filters: this.filters
        }, params, {
          filters: this.filters
        })

        if (this.issues === null) {
          this.tableData = []
        } else {
          this.tableData = this.issues.slice(0).filter((issue) => {
            return typeof issue !== 'undefined'
          }).filter((issue) => {
            return this.groupFilter === '' || issue.beacon.group !== null && issue.beacon.group.name === this.groupFilter
          }).filter((issue) => {
            return issue.beacon.name.toLowerCase().includes(this.search.toLowerCase())
          }).filter((issue) => {
            return this.statusFilter === this.ISSUES_FILTER_ALL
                || (this.statusFilter === this.ISSUES_FILTER_OPEN && !issue.resolved)
                || (this.statusFilter === this.ISSUES_FILTER_CLOSED && issue.resolved)
          })
        }

        let newMapData = [];
        this.tableData.forEach(tableIssue => {
          if(!newMapData.find(mapBeacon => mapBeacon.id == tableIssue.beacon.id))
            newMapData.push(tableIssue.beacon)
        })

        this.$set(this, 'mapData', newMapData.slice(0))

        this.tableData.sort((beaconA, beaconB) => {
          let valA = beaconA
          let valB = beaconB
          let sortingCols = params.sorting.col.split('.')
          for(let i = 0; i < sortingCols.length; i++) {
            valA = valA != null? valA[sortingCols[i]]: null
            valB = valB != null? valB[sortingCols[i]]: null
          }
          if(valA != null && valB == null) {
            return -1
          }
          if(valA == null && valB != null) {
            return 1
          }
          if (valA < valB) {
            return params.sorting.order === 'asc' ? -1 : 1
          }
          if (valA > valB) {
            return params.sorting.order === 'asc' ? 1 : -1
          }
          return 0
        })

        this.tableMeta.totalRecords = this.tableData.length
        params.pagination.page = Math.min(Math.max(params.pagination.page, 1), Math.ceil(this.tableMeta.totalRecords / this.tableMeta.pagination.records))

        let currentIndex = (params.pagination.page - 1) * params.pagination.records
        let nextIndex = params.pagination.page * params.pagination.records
        this.tableData = this.tableData.slice(currentIndex, nextIndex)
        this.tableMeta.sorting.col = params.sorting.col
        this.tableMeta.sorting.order = params.sorting.order
      },
      iconSvg(beacon) {
        let uri = location.origin;
        switch (beacon.status) {
          case 'BATTERY_LOW':
          case 'ISSUE':
            uri += require('../assets/img/map/map_icon_issue.svg')
            break
          case 'CONFIGURATION_PENDING':
            uri += require('../assets/img/map/map_icon_pending.svg')
            break
          case 'UNKNOWN_STATUS':
            uri += require('../assets/img/map/map_icon_unknownstatus.svg')
            break
          case 'NOT_ACCESSIBLE':
            uri += require('../assets/img/map/map_icon_notaccessible.svg')
            break
          default:
            uri += require('../assets/img/map/map_icon_ok.svg')
            break
        }

        return encodeURI(uri);
      },
      resetFilter() {
        this.map.setView([46.6568142, 11.423318], 9);
        this.search = ''
        this.groupFilter = ''
      },
      reload() {
        this.fetchIssues()
        this.fetchGroups()
      },
    },
    async mounted() {
      this.loaded = false
      this.$nextTick()
      this.clear().then(() => {
        this.$set(this, 'loaded', false)
      })
      try {

        this.search = sessionStorage.getItem('issues_search') || ''
        this.groupFilter = sessionStorage.getItem('group_filter') || ''

        this.statusFilter = sessionStorage.getItem('issues_status_filter') || this.ISSUES_FILTER_OPEN

        this.L = await initMap();
        this.map = this.L.map('map')
        this.map.zoomControl.setPosition('topright')
        let map = this.map

        this.map.on('zoomend', function() {
          sessionStorage.setItem('map_zoom', map.getZoom())
        });

        this.map.on('moveend', function(e) {
          sessionStorage.setItem('map_lat', e.target.getCenter().lat)
          sessionStorage.setItem('map_lon', e.target.getCenter().lng)
        });

        // get previous zoom
        let prevZoom = sessionStorage.getItem('map_zoom') || 9
        let prevLat  = sessionStorage.getItem('map_lat')  || 46.6568142
        let prevLon  = sessionStorage.getItem('map_lon')  || 11.423318

        // setView after on zoomend/moveend so that they fire the first time
        this.map.setView([prevLat, prevLon], prevZoom);

        this.L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
          attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
        }).addTo(this.map);

        this.fetchIssues()
        this.fetchGroups()


      } catch (error) {
        window.console.log(error);
        throw error;
        // this.loaded = true
      }
    },
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss" scoped>
  @import '../variables';
  h1 {
    display: inline-block;
  }

  .issues-table-container {
    margin-top: 8em !important;
    @media only screen and (min-width: 576px) {
      margin-top: 4em !important;
    }
  }

  .issues-container {
    margin-top: 4em !important;
  }

  .issue-map {
    width: 100%;
    height: 100%;
    z-index: 0;
  }

  #view-switch {
    z-index: 1000;
  }

  .btn-view-switch {
    color: white;
    background: $grey;
    z-index: 1000;
    border-radius: 0.5em;
    opacity: 0.7;
    height: 2.25em;
    line-height: 1.125em;
    padding: 0 0.7em;

    &:hover {
      color: white;
      opacity: 0.9;
    }

    &:focus {
      box-shadow: none;
    }

    &.btn-view-switch-active {
      background: $status-blue;
      opacity: 1;

      &:hover {
        background: $background-blue;
      }
    }
  }

  .add-fab {
    position: absolute;
    bottom: 0;
    right: 1em;
    transform: translateY(50%);
    border-radius: 50%;
    background-image: url("../assets/ic_add_beacon.svg");
  }

  .beacon-display {
    position: relative;
  }

  .btn-delete {
    mask-image: url("../assets/delete.svg");
    background-color: black;

    &:hover {
      background-color: red;
    }
  }


  .btn-reset {
    background: $light-blue;
    border-color: $light-blue;
    font-size: 0.8rem;
    color: white;

    &:hover {
      background: $lighter-blue;
      border-color: $lighter-blue;
      color: white;
    }
  }

  .table-header {
    background-color: $background-grey;
    color: $grey;
    padding-top: 1em;
    padding-bottom: 1em;
    font-size: 0.8rem;
  }

  .beacon-row {
    font-size: 0.8rem;
    font-weight: lighter;
  }

  .battery {
    height: 1.5em;
    width: 0.75em;
    background: $lighter-grey;
    position:relative;

    &:before,
    &:after {
      z-index:10;
      content:'';
      display:block;
      position:absolute;
      background: white;
      width: 0.2em;
      height: 0.15em;
      top:0;
    }
    &:before {
      left: 0;
    }
    &:after {
      right: 0;
    }
    .chargestatus {
      position:absolute;
      top: 0;
      left:0;
      right:0;
      background: grey;
    }

    &.warning {
      background: lighten($status-yellow, 35%);
      .chargestatus {
        background: lighten($status-yellow, 25%);
      }
    }

    &.danger {
      background: darkred;
      .chargestatus {
        background: red;

      }
    }
  }

</style>
