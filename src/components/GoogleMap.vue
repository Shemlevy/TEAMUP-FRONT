<template>
  <section> 
    <input class="controls" style="z-index:999;" type="text" @change="getGeoByAddress" placeholder="Search place for a New Game" ref="googleSearch">    
    <div class="google-map" :id="mapName"></div>
    <button ref="locationBtn" class="location-btn" @click="getUserLoc"><i title="Get my location location-btn" class="material-icons">my_location</i></button>
   <transition name="slide-fade">
      <cg-popup v-if="show" @close-dialog="close"></cg-popup>
    </transition>
  </section>
</template>
<script>
import {
  GET_SELCTED_GAME,
  SET_SELECTED_GAME,
  GET_GAMES
} from "../store/modules/game/Game.module";
import MapService from "../service/map/MapService";
import CgPopup from "../components/CgPopup";
import {
  SET_CURR_ADDRESS,
  GET_PICK_ADDRESS,
  SET_USER_LOCATION
} from "../store/modules/map/Map.module";
import EventBusService, { GET_LOCATION } from "../service/EventBusService";

export default {
  name: "google-map",
  data: function() {
    return {
      // currAddress: null,
      show: false,
      mapName: this.name + "-map",
      map: null,
      bounds: null,
      tempMarker: null,
      infoWindow: null,
      // markers: [],
      searchBox: null
      // searchLocation: {
      //   latitude: 51.501527,
      //   longitude: -0.1921837
      // },
    };
  },
  created() {
    EventBusService.$on(GET_LOCATION, () => {
      this.getUserLoc();
    });
    var markerIcon = {
      url: "../../static/icons/addPlace.png",
      scaledSize: new google.maps.Size(40, 40)
    };
    this.tempMarker = new google.maps.Marker({
      icon: markerIcon,
      animation: google.maps.Animation.DROP,
      draggable: false,
      map: this.map
    });
    let self = this;
    this.tempMarker.addListener("click", function(e) {
      self.show = !self.show;
    });
  },
  mounted() {
    // this.renderMap();
  },
  watch: {
    selctedGame() {
      if (!this.selctedGame) return;
      var lat = this.selctedGame.location.lat;
      var lng = this.selctedGame.location.lng;
      this.map.panTo(new google.maps.LatLng(lat, lng));
      this.map.setZoom(18);
    },
    games() {
      this.renderMap();
    },
    pickLatLng() {
      this.map.panTo(
        new google.maps.LatLng(this.pickLatLng.lat, this.pickLatLng.lng)
      );
    }
  },
  computed: {
    selctedGame() {
      return this.$store.getters[GET_SELCTED_GAME];
      // return this.$store.getters.GET_SELCTED_GAME;
    },
    games() {
      return this.$store.getters[GET_GAMES];
    },
    pickLatLng() {
      return this.$store.getters[GET_PICK_ADDRESS];
    }
  },
  methods: {
    renderMap() {
      let self = this;
      this.bounds = new google.maps.LatLngBounds();
      const element = document.getElementById(this.mapName);

      let eltCenter = [{ location: { lat: 32.072634, lng: 34.763987 } }];
      const mapCentre = this.games ? this.games[0] : eltCenter[0];

      const options = {
        center: new google.maps.LatLng(
          mapCentre.location.lat,
          mapCentre.location.lng
        )
      };
      
      this.map = new google.maps.Map(element, { zoom : 18 });
      this.infoWindow = new google.maps.InfoWindow();

      this.tempMarker.setMap(this.map);
      var input = this.$refs.googleSearch;
      var locationBtn = this.$refs.locationBtn;

      this.map.controls[google.maps.ControlPosition.TOP_RIGHT].push(input);
      this.map.controls[google.maps.ControlPosition.RIGHT].push(locationBtn);
      this.searchBox = new google.maps.places.SearchBox(input);

      if (this.games) {
        this.games.forEach(coord => {
          const position = new google.maps.LatLng(
            coord.location.lat,
            coord.location.lng
          );
          var markerIcon = {
            url: "../../static/icons/gameMarker.png",
            scaledSize: new google.maps.Size(40, 40)
          };

          const marker = new google.maps.Marker({
            icon: markerIcon,
            animation: google.maps.Animation.DROP,
            position,
            map: this.map
          });
          marker.addListener("click", e => {
            this.$router.push(`/game/${coord._id}`);
          });
          // this.markers.push(marker)
          this.map.fitBounds(this.bounds.extend(position));
        });
      } else {
        this.map.fitBounds(this.bounds.extend(options.center));
      }
    },
    getGeoByAddress(e) {
      let pos = e.target.value
      MapService.getGeoByAddress(pos).then(res => {
        // let pos = new google.maps.LatLng(res.lat, res.lng);
        this.map.setZoom(16);
        this.map.panTo(new google.maps.LatLng(res.lat, res.lng));

        this.$store.commit({
          type: SET_CURR_ADDRESS,
          address: res
        });
        this.tempMarker.setPosition(res.postion);
      });
    },
    close() {
      this.show = false;
    },
    getUserLoc() {
      let self = this;
      if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(
          function(position) {
            var pos = {
              lat: position.coords.latitude,
              lng: position.coords.longitude
            };

            self.$store.commit({
              type: SET_USER_LOCATION,
              location: pos
            });
            self.infoWindow.setPosition(pos);
            self.infoWindow.setContent("You are Here");
            self.infoWindow.open(self.map);
            self.map.setCenter(pos);
          },
          function() {
            handleLocationError(true, self.infoWindow, self.map.getCenter());
          }
        );
      } else {
        // Browser doesn't support Geolocation
        handleLocationError(false, self.infoWindow, map.getCenter());
      }
    },
    handleLocationError(browserHasGeolocation, infoWindow, pos) {
      self.infoWindow.setPosition(pos);
      self.infoWindow.setContent(
        browserHasGeolocation
          ? "Error: The Geolocation service failed."
          : "Error: Your browser doesn't support geolocation."
      );
      self.infoWindow.open(self.map);
    }
  },
  components: {
    CgPopup
  }
};
</script>





<style scoped>
/* section{
  position: absolute;
} */
.google-map {
  width: 100%;
  height: 600px;
}
.controls {
  width: 240px;
  color: var(--third-color);
  height: 25px;
  font-size: 1.4em;
  font-weight: 500;
  margin: 13px;
  border-radius: 5px;
  padding: 8px;
  z-index: 5;
}

.slide-fade-enter-active {
  transition: all 0.5s ease;
}
.slide-fade-leave-active {
  transition: all 0.4s cubic-bezier(1, 0.5, 0.8, 1);
}
.slide-fade-enter,
.slide-fade-leave-to {
  transform: translateY(50px);
  opacity: 0;
}

.location-btn {
  border: 0px;
  margin: -35px 44px;
  /* padding: 0px; */
  cursor: pointer;
  /* user-select: none; */
  text-align: center;
  width: 25px;
  height: 25px;
  overflow: hidden;
  color: gray;
  background-color: white;
}
.material-icons {
  margin: 0;
}
</style>
