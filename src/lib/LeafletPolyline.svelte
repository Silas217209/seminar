<script>
    import L from 'leaflet';
    import { getContext } from "svelte";

    export let latlongs = [[52.76031, 13.646773333333332], [52.76031, 13.646773333333332]];

    let map = getContext('leafletMapInstance');

    var polyline = L.polyline(latlongs, {color: 'red', smoothFactor: 1.5}).addTo(map);
    map.setView(polyline.getCenter(), 13);
</script>