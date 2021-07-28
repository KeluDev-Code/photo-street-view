<template>
  <div style="width: 100%; height: 100vh;">
    <!-- <input id="pac-input" class="controls" type="text"
     placeholder="Search Box"> -->
    <!-- <q-input id="pac-input" type="text" class="controls" dense
    :label="$t('Fields.direction')" /> -->
    <div id="map" />
    <div id="pano"></div>
  </div>
</template>

<script lang="ts">
import {
  defineComponent, onUnmounted, onMounted,
} from 'vue';
// import {} from 'googlemaps';
declare global {
    interface Window { initAutocomplete: ()=> void }
}
export default defineComponent({
  name: 'GoogleMap',
  props: {
  },
  setup() {
    if (!('geolocation' in navigator)) {
      // errorStr = 'Geolocation is not available.';

    }

    const googleApiKey = 'AIzaSyDiSx1-u4wn1iTpSOVne_NpBo5SfI3eOjE';

    // Santo Domingo RD
    let latitude = 18.4867;
    let longitude = -69.9312;

    let { google } = window;
    let map: google.maps.Map;
    let geocoder: google.maps.Geocoder;
    const markers: google.maps.Marker[] = [];
    let input: HTMLInputElement | undefined;

    const setMapOnAll = (currentMap: google.maps.Map | null) => {
      for (let i = 0; i < markers.length; i++) {
        markers[i].setMap(currentMap);
      }
    };

    // Removes the markers from the map, but keeps them in the array.
    const clearMarkers = () => {
      setMapOnAll(null);
    };

    const getGeocode = (latLng: google.maps.LatLng) => {
      geocoder.geocode({
        location: latLng,
      }, (results, status) => {
        if (status === google.maps.GeocoderStatus.OK) {
          if (results && results[0]) {
            const components = results[0];
            if (input) {
              input.value = components.formatted_address;
            }

            clearMarkers();
            const marker = new google.maps.Marker({
              position: latLng,
              map,
            });
            markers.push(marker);
          }
        }
      });
    };

    const initMap = () => {
      const mapElement = document.getElementById('map');

      if (mapElement) {
        google = window.google;
        geocoder = new google.maps.Geocoder();
        map = new google.maps.Map(mapElement, {
          center: { lat: latitude, lng: longitude },
          zoom: 13,
          mapTypeId: google.maps.MapTypeId.ROADMAP,
          disableDefaultUI: true,
        });
        const streetViewElement = document.getElementById('pano');
        let panorama: google.maps.StreetViewPanorama;
        if (streetViewElement) {
          panorama = new google.maps.StreetViewPanorama(
            streetViewElement,
            {
              showRoadLabels: false,
              disableDefaultUI: true,
              position: {
                lat: latitude,
                lng: longitude,
              },
              pov: {
                heading: 34,
                pitch: 10,
              },
            },
          );
          map.setStreetView(panorama);
        }

        const defaultMarker = new google.maps.Marker({
          position: { lat: latitude, lng: longitude },
          map,
        });
        markers.push(defaultMarker);

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

        navigator.geolocation.getCurrentPosition((pos) => {
          latitude = pos.coords.latitude;
          longitude = pos.coords.longitude;

          const hotspotLoc = {
            lat: latitude,
            lng: longitude,
          };
          map.setCenter(hotspotLoc);

          if (streetViewElement) {
            panorama = new google.maps.StreetViewPanorama(
              streetViewElement,
              {
                showRoadLabels: false,
                disableDefaultUI: true,
                position: {
                  lat: latitude,
                  lng: longitude,
                },
                pov: {
                  heading: 34,
                  pitch: 10,
                },
              },
            );
            map.setStreetView(panorama);
          }
          markers.push(new google.maps.Marker({
            map,
            position: hotspotLoc,
          }));

          setTimeout(() => {
            getGeocode(new google.maps.LatLng(latitude, longitude));
          }, 100);
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

    return {
    };
  },
});

</script>

<style>
  /* Always set the map height explicitly to define the size of the div
    * element that contains the map. */

#map {
  height: 45%;
  width: 100%;
}
#pano {
  height: 50%;
  width: 100%;
}

  /* Optional: Makes the sample page fill the window. */
  html, body {
    height: 100%;
    margin: 0;
    padding: 0;
  }
  #description {
    font-family: Roboto;
    font-size: 15px;
    font-weight: 300;
  }

  #infowindow-content .title {
    font-weight: bold;
  }

  #infowindow-content {
    display: none;
  }

  #map #infowindow-content {
    display: inline;
  }

  .pac-card {
    margin: 10px 10px 0 0;
    border-radius: 2px 0 0 2px;
    box-sizing: border-box;
    -moz-box-sizing: border-box;
    outline: none;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.3);
    background-color: #fff;
    font-family: Roboto;
    z-index: 500000;
  }

  /* #pac-container {
    padding-bottom: 12px;
    margin-right: 12px;
    z-index: 500000;
  } */
  .pac-container {
    background-color: #FFF;
    z-index: 10000;
    position: fixed;
    display: inline-block;
    float: left;
    width: 500px;
}
  .pac-controls {
    display: inline-block;
    padding: 5px 11px;
     z-index: 500000;
  }

  .pac-controls label {
    font-family: Roboto;
    font-size: 13px;
    font-weight: 300;
     z-index: 500000;

  }

  #pac-input {
    background-color: #fff;
    font-family: Roboto;
    font-size: 15px;
    font-weight: 300;
    /* margin-left: 12px; */
    padding: 0 11px 0 13px;
    text-overflow: ellipsis;
    width: 540px;
    min-width: 340px;

  }

  #pac-input:focus {
    border-color: #4d90fe;
  }

  #title {
    color: #fff;
    background-color: #4d90fe;
    font-size: 25px;
    font-weight: 500;
    padding: 6px 12px;
  }
  #target {
    width: 345px;
  }
</style>
