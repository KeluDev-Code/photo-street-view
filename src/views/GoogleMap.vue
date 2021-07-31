<template>
  <div class="column" :class="{'fullPano': fullPano}" style="width: 100%; height: 100vh;"
       @keyup.esc="changeFullStreet()"
  >
    <div class="row justify-center absolute-left">
      <div class="ma-sm">
        <input v-model="inputUrl" type="text" placeholder="Search Box" style="height: 17px" @keyup.enter="processURL()">
        <button @click="processURL()">
          &#128269;
        </button>
        <button @click="takeScreenShot()">
          &#128248;
        </button>
      </div>
    </div>
    <div class="col-4" :class="{'hide': fullPano}">
      <div id="map" />
    </div>
    <div class="col relative">
      <div class="btn-absolute">
        <button style="font-weight: bold; font-size: 16px;" @click="changeFullStreet()">
          {{ fullPano ? '&#709;' : '&#708;' }}
        </button>
      </div>
      <div id="pano" />
      <!-- https://html2canvas.hertzen.com/ -->
    </div>
  </div>
</template>

<script lang="ts">
import {
  defineComponent, onUnmounted, onMounted, ref,
} from 'vue';
import html2canvas from 'html2canvas';

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

    const changeFullStreet = () => {
      fullPano.value = !fullPano.value;
      setTimeout(() => {
        google.maps.event.trigger(panorama, 'resize');
      }, 100);
    };
    // https://www.aspsnippets.com/Articles/Take-Screenshot-Snapshot-of-Google-Maps-using-JavaScript.aspx
    const takeScreenShot = () => {
      // const canvaStreeView = document.getElementsByClassName('widget-scene-canvas');
      const canvaStreeView = document.getElementById('pano');
      if (canvaStreeView) {
        html2canvas(canvaStreeView, {
          useCORS: true,
        }).then((canvas) => {
          const link = document.createElement('a');

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
          }
        });
        /* const newCanvas = document.createElement('canvas');
        newCanvas.id = 'screenShot';
        const context = newCanvas.getContext('2d');

        // set dimensions
        newCanvas.width = (canvaStreeView[0] as HTMLCanvasElement).width;
        newCanvas.height = (canvaStreeView[0] as HTMLCanvasElement).height;

        // apply the old canvas to the new one
        context!.drawImage(canvaStreeView[0] as HTMLCanvasElement, 0, 0);
        // const canvas = canvaStreeView[0].cloneNode(true);
        // console.log(canvas);
        // (canvas as Element).className = '';

        document.body.appendChild(newCanvas);
        /// html2canvas(body[0]).then((canvas) => {
        /* (canvaStreeView[0] as HTMLCanvasElement).toBlob((blob) => {
          // eslint-disable-next-line @typescript-eslint/no-explicit-any
            const file: any = blob;
            file.lastModifiedDate = new Date();
            file.name = 'Capture';
          }); *
        const fileName = 'Capture';
        const blob = (newCanvas as HTMLCanvasElement).toDataURL();
        if (blob) {
          try {
            if (window.navigator.msSaveOrOpenBlob) {
              window.navigator.msSaveOrOpenBlob(blob, `${fileName}`);
            } else {
              const downloadUrl = blob;
              const link = window.document.createElement('a');
              // safari doesn't support this yet
              if (typeof link.download === 'undefined') {
                window.location.href = downloadUrl;
              } else {
                link.href = downloadUrl;
                link.setAttribute('download', `${fileName}.png`);
                window.document.body.appendChild(link);
                link.click();
              }

              try {
                setTimeout(() => { window.document.body.removeChild(link); }, 200);
                // eslint-disable-next-line no-empty
              } catch (err) { }
            }
            // eslint-disable-next-line no-empty
          } catch (ex) { }
        } */
        // });
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

    return {
      inputUrl,
      processURL,
      fullPano,
      changeFullStreet,
      takeScreenShot,
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
  top: -12px;
  left: 50%;
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

</style>
