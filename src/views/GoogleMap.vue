<template>
  <div class="column" :class="{'fullPano': fullPano}" style="width: 100%; height: 100vh;"

       @mouseenter="changeHideGUI(false)" @mouseleave="changeHideGUI(true)"
  >
    <div class="row justify-center absolute-left" :class="{'hide': hideGUI}">
      <div class="row ma-sm">
        <input v-model="inputUrl" type="text" placeholder="Search Box" class="search-box" @keyup.enter="processURL()">
        <button class="icon" @click="processURL()">
          <svg width="24" height="24" viewbox="0 0 24 24">
            <path fill="#ffffff" :d="mdiMagnify" />
          </svg>
        </button>
      </div>
    </div>
    <div class="col-4" :class="{'hide': fullPano}">
      <div id="map" />
    </div>
    <div class="col relative">
      <div class="btn-absolute" :class="{'hide': hideGUI}">
        <button class="icon icon-md" @click="changeFullStreet()">
          <svg width="24" height="22" viewbox="0 0 24 22" style="transform: scale(1.5);">
            <path fill="#ffffff" :d="fullPano ? mdiChevronDown : mdiChevronUp" />
          </svg>
        </button>
      </div>
      <!-- https://html2canvas.hertzen.com/ -->
      <div id="pano" />
    </div>
  </div>
</template>

<script lang="ts">
import {
  defineComponent, onUnmounted, onMounted, ref,
} from 'vue';
import { mdiMagnify, mdiChevronUp, mdiChevronDown } from '@mdi/js';

declare global {
    interface Window { initAutocomplete: ()=> void }
}
export default defineComponent({
  name: 'GoogleMap',
  setup() {
    const fullPano = ref(false);
    const hideGUI = ref(false);
    const inputUrl = ref('');
    if (!('geolocation' in navigator)) {
      // errorStr = 'Geolocation is not available.';

    }

    const googleApiKey = process.env.VUE_APP_GOOGLE_MAPS_API_KEY;

    // Manchester
    let latitude = 53.4730789;
    let longitude = -2.2482232;

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

    // TODO: buscar lloc
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
        }, () => { });
      }
    };

    // https://www.aspsnippets.com/Articles/Take-Screenshot-Snapshot-of-Google-Maps-using-JavaScript.aspx
    /* const takeScreenShot = () => {
      // const [canvaStreeView] = document.getElementsByClassName('widget-scene-canvas');
      const canvaStreeView = document.getElementById('pano');
      if (canvaStreeView) {
        /* let img = (canvaStreeView as HTMLCanvasElement).toDataURL('image/png');
        img = img.replace('data:image/png;base64,', '');
        const finalImageSrc = `data:image/png;base64,${img}`;
        const link = document.createElement('img');

        link.src = finalImageSrc;
        document.body.appendChild(link); *
        html2canvas(canvaStreeView, {
          useCORS: true,
        }).then((canvas) => {
          const img = canvas.toDataURL().replace('data:image/png;base64,', '');
          const finalImageSrc = `data:image/png;base64,${img}`;
          const link = document.createElement('img');

          link.src = finalImageSrc;
          document.body.appendChild(link);
          /* const link = document.createElement('a');

          if (typeof link.download === 'string') {
            link.href = canvas.toDataURL();
            link.download = 'Capture.png';

            // Firefox requires the link to be in the body
            document.body.appendChild(link);

            // simulate click
            link.click();

            // remove the link when done
            document.body.removeChild(link);
          } else {
            window.open(canvas.toDataURL());
          } *
        });
      }
    }; */

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

    const changeHideGUI = (hide: boolean) => {
      if (fullPano.value) {
        hideGUI.value = hide;
      }
    };

    return {
      inputUrl,
      processURL,
      fullPano,
      changeFullStreet,
      changeHideGUI,
      hideGUI,

      mdiMagnify,
      mdiChevronUp,
      mdiChevronDown,
    };
  },
});

</script>

<style>
  /* Always set the map height explicitly to define the size of the div
    * element that contains the map. */
html,
body {
  height: 100%;
  width: 100%;
  margin: 0;
  padding: 0;
}
@media print {
  img {
    max-width: 100% !important;
    }
  #pano {
    height: 200px !important;
    width: 200px !important;
  }
}

#map {
  height: 100%;
  width: 100%;
}

#pano {
  height: 100%;
  width: 100vw;
}

.column,
.row {
  display: flex;
  flex-wrap: wrap;
}

.column {
  flex-direction: column;
}

.column > .col {
  height: auto;
  max-height: 100%;
  min-height: 0;
  flex: 10000 1 0%;
}

.col-auto {
  flex: 0 0 auto !important;
}

.column > .col-8 {
  height: 66.6667%;
  width: auto;
}

.column > .col-4 {
  height: 33.3333%;
  width: auto;
}

.relative {
  position: relative;
}

.absolute-left {
  position: absolute;
  top: 15px;
  left: 10px;
  z-index: 10;
}

.btn-absolute {
  position: absolute;
  top: -22px;
  left: 0;
  right: 0;
  margin-left: auto;
  margin-right: auto;
  z-index: 10;
}

.btn-absolute button {
  width: 50px;
}

.fullPano .btn-absolute {
  top: 1px;
}

.justify-center {
  justify-content: center;
}

.ma-sm {
  margin: 10px 15px;
}

.height-50p {
  height: 50%;
}

.full-screen {
  width: 100vw !important;
  height: 100vh !important;
}

.hide {
  display: none;
}

.search-box {
  padding: 1px 2px;
  margin: 8px 0;
  height: 28px;
  border: 1px solid #00897B;
  border-radius: 4px;
}

.search-box:focus {
  outline: none !important;
  border: 1px solid #26A69A;
  box-shadow: 0 0 10px #26A69A;
}

button.icon {
  padding: 0;
  margin: 8px;
  width: 32px;
  height: 32px;
  border: 1px solid #004D40;
  border-radius: 4px;
  background-color: #00897B;
  text-align: center;
  transition: all .3s;
  cursor: pointer;
}

button.icon-md {
  width: 48px;
  height: 32px;
}

button.icon:hover {
  background-color: #26A69A;
}

button.icon:active {
  transform: scale(.88);
}

button.icon svg {
  padding: 0;
  margin: 0;
}

</style>
