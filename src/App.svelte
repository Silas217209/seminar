<script lang="ts">
  $: coordslist = [];
  let id
  let distancelist = [];
  let accuracy
  const watchposition = () => {
    id = navigator.geolocation.watchPosition(success, error, {
      enableHighAccuracy: true,
      maximumAge: 1000,
      timeout: 10000,
    });
  };
  const success = (location: { coords: { latitude: any; longitude: any; accuracy: any; }; }): void => {
    coordslist.push([location.coords.latitude, location.coords.longitude]);
    accuracy = location.coords.accuracy
    try {
      distancelist = distancelist.concat([
        getDistance(
          location.coords.latitude,
          location.coords.longitude,
          coordslist[coordslist.length - 2][0],
          coordslist[coordslist.length - 2][1]
        ),
      ]);
    } catch {
      distancelist = distancelist.concat([0]);
    }
  };
  const error = (error: any) => {
    distancelist = [error];
    console.log(error);
  };

  function getDistance(lat1, lon1, lat2, lon2) {
    var R = 6371; // Radius of the earth in km
    var dLat = deg2rad(lat2 - lat1); // deg2rad below
    var dLon = deg2rad(lon2 - lon1);
    var a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(deg2rad(lat1)) *
        Math.cos(deg2rad(lat2)) *
        Math.sin(dLon / 2) *
        Math.sin(dLon / 2);
    var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
    var d = R * c * 1000; // Distance in m
    return d;
  }

  function deg2rad(deg) {
    return deg * (Math.PI / 180);
  }
</script>

<main>
  <button on:click={() => watchposition()}>Watch position</button>
  <pre><code>{accuracy}</code></pre>
</main>

<style>
  .logo {
    height: 6em;
    padding: 1.5em;
    will-change: filter;
  }
  .logo:hover {
    filter: drop-shadow(0 0 2em #646cffaa);
  }
  .logo.svelte:hover {
    filter: drop-shadow(0 0 2em #ff3e00aa);
  }
  .read-the-docs {
    color: #888;
  }
</style>
