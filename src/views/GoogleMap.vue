<template>
  <div class="column" style="width: 100%; height: 100vh;">
    <div class="row justify-center col col-auto">
      <div class="ma-sm">
        <input v-model="inputUrl" type="text" placeholder="Search Box" @keyup.enter="processURL()">
        <button @click="processURL()">
          Search
        </button>
      </div>
      <div class="ma-sm">
        <button @click="changeFullStreet()">
          Full Street View
        </button>
      </div>
    </div>
    <div class="col" :class="{'hide': fullPano}">
      <div id="map" />
    </div>
    <div class="col-8">
      <div id="pano" />
    </div>
  </div>
</template>

<script lang="ts">
import {
  defineComponent, onUnmounted, onMounted, ref,
} from 'vue';

declare global {
    interface Window { initAutocomplete: ()=> void }
}
export default defineComponent({
  name: 'GoogleMap',
  props: {
  },
  setup() {
    const fullPano = ref(false);
    const inputUrl = ref('');
    if (!('geolocation' in navigator)) {
      // errorStr = 'Geolocation is not available.';

    }

    const googleApiKey = process.env.VUE_APP_GOOGLE_MAPS_API_KEY;

    // Manchester
    let latitude = 53.4754446;
    let longitude = -2.2615203;

    let { google } = window;
    let map: google.maps.Map;
    let panorama: google.maps.StreetViewPanorama;
    let streetViewElement: HTMLElement | null;
    const markers: google.maps.Marker[] = [];

    // Removes the markers from the map, but keeps them in the array.
    const clearMarkers = () => {
      for (let i = 0; i < markers.length; i++) {
        markers[i].setMap(null);
      }
    };

    const centerMap = (latLng: google.maps.LatLng) => {
      map.setCenter(latLng);

      if (streetViewElement) {
        panorama = new google.maps.StreetViewPanorama(
          streetViewElement,
          {
            showRoadLabels: false,
            disableDefaultUI: true,
            position: latLng,
            zoom: 1.5,
          },
        );
        map.setStreetView(panorama);
      }

      clearMarkers();
      markers.push(new google.maps.Marker({
        map,
        position: latLng,
      }));
    };

    const processURL = () => {
      const uri = decodeURIComponent(inputUrl.value);
      const match = uri.match('^https://.*!1s(.*)!2e.*$');
      let id = match && match[1];
      let latlng = ['None', 'None'];
      latlng = uri.match(/@([\d]*\.[\d]*),(-?[\d]*\.[\d]*),/) || [];
      const pov = uri.match(/,([0-9.]+)y,([0-9.]+)h,([0-9.]+)t/);

      if (id) {
        if (id[0] === '-') {
          id = `F:${id}`;
        }

        if (pov) {
          latitude = Number(latlng[1]);
          longitude = Number(latlng[2]);
          const heading = Number(pov[2]);
          const pitch = Number(pov[3]);

          centerMap(new google.maps.LatLng(latitude, longitude));

          panorama.setPano(id);
          panorama.setPov({
            heading,
            pitch: pitch - 90,
          });
        }
      } else {
        panorama.setPano('');
      }
    };

    const initMap = () => {
      const mapElement = document.getElementById('map');
      streetViewElement = document.getElementById('pano');

      if (mapElement) {
        google = window.google;
        map = new google.maps.Map(mapElement, {
          center: { lat: latitude, lng: longitude },
          zoom: 8,
          mapTypeId: google.maps.MapTypeId.ROADMAP,
          disableDefaultUI: true,
        });

        centerMap(new google.maps.LatLng(latitude, longitude));

        map.addListener('click', (mapsMouseEvent: {latLng: google.maps.LatLng}) => {
          centerMap(mapsMouseEvent.latLng);
        });

        navigator.geolocation.getCurrentPosition((pos) => {
          latitude = pos.coords.latitude;
          longitude = pos.coords.longitude;
          centerMap(new google.maps.LatLng(latitude, longitude));

          /* input = document.getElementById('pac-input') as HTMLInputElement;
        if (input) {
          const searchBox = new google.maps.places.SearchBox(input);
          map.controls[google.maps.ControlPosition.TOP_LEFT].push(input);
          // Bias the SearchBox results towards current map's viewport.
          map.addListener('bounds_changed', () => {
            const bounds = map.getBounds();
            if (bounds) {
              searchBox.setBounds(bounds);
            }
          });
        } */
          /* setTimeout(() => {
            getGeocode(new google.maps.LatLng(latitude, longitude));
          }, 100); */
        }, () => { });
      }
    };

    window.initAutocomplete = () => {
      initMap();
    };

    onMounted(() => {
      let gMaps = document.getElementById('googleapis');
      if (!gMaps) {
        gMaps = document.createElement('script');
        gMaps.setAttribute('id', 'googleapis');

        gMaps.setAttribute('type', 'text/javascript');
        gMaps.setAttribute('src', `https://maps.googleapis.com/maps/api/js?key=${googleApiKey}&libraries=places&callback=initAutocomplete`);
        const div = document.getElementById('divGoogle');
        if (div) {
          div.appendChild(gMaps);
        }
        document.head.appendChild(gMaps);
      } else {
        initMap();
      }
    });

    onUnmounted(() => {
      try {
        const gMaps = document.getElementById('divGoogle');
        if (gMaps) document.removeChild(gMaps);

        document.querySelectorAll('.pac-container').forEach((a) => {
          a.remove();
        });

        // eslint-disable-next-line no-empty
      } catch (err) { }
    });
    const changeFullStreet = () => {
      fullPano.value = !fullPano.value;
      setTimeout(() => {
        google.maps.event.trigger(panorama, 'resize');
      }, 100);
    };

    return {
      inputUrl,
      processURL,
      fullPano,
      changeFullStreet,
    };
  },
});

</script>

<style>
  /* Always set the map height explicitly to define the size of the div
    * element that contains the map. */
html, body {
    height: 100%;
    margin: 0;
    padding: 0;
  }

#map {
  height: 100%;
  width: 100%;
}
#pano {
  height: 100%;
  width: 100%;
}
.column, .row{
  display: flex;
  flex-wrap: wrap;
}
.column{
  flex-direction: column;
}
.column > .col{
  height: auto;
  max-height: 100%;
  min-height: 0;
  flex: 10000 1 0%;
}
.col-auto{
  flex: 0 0 auto !important;
}
.column > .col-8 {
    height: 66.6667%;
    width: auto;
}
.justify-center {
    justify-content: center;
}
.ma-sm{
  margin: 10px 15px;
}
.height-50p {
  height: 50%;
}
.full-screen {
  width: 100vw !important;
  height: 100vh !important;
}
.hide{
  display: none;
}
</style>
