<script lang="ts">

    ///////////////////   Initialisation   ///////////////////
        import KalmanFilter from 'kalmanjs'
        import LeafletMap from "./lib/LeafletMap.svelte";
        import LeafletPolyline from "./lib/LeafletPolyline.svelte";


        let coords = []
        let speed = 0
        let fspeed = 0
        let accuracy = 0
        let coordslist = [];
        let id: number
        let distancelist = [];
        let fdistancelist = [];
        let timeslist = [];
        let db: IDBDatabase
        let textinput = ""
        let kfspeed = new KalmanFilter
        let kfdistance = new KalmanFilter
        let iswatching = false
        let lat: number
        let long: number
        

    ///////////////////   storage Initialisation   ///////////////////

        // opening indexed db version 4 for location storage
        const request = window.indexedDB.open("Data", 4);

        // nessesary indexed db functions
        request.onerror = event => {
            console.log("error: ");
        };
        request.onsuccess = event => {
            db = request.result

        };
        request.onupgradeneeded = function (event) {
            var db = request.result
            db.createObjectStore("data", {keyPath: "id", autoIncrement: true})
        };



    ///////////////////   position capture   ///////////////////

        // one time manual requested position
        function getpos() {
            navigator.geolocation.getCurrentPosition((location) => {
                accuracy = location.coords.accuracy
                lat = location.coords.latitude
                long = location.coords.longitude
            }, error, {
                enableHighAccuracy: true,
                maximumAge: 50,
            })
        }

        // position tracking intitialisation
        const watchposition = () => {
            if (!iswatching) {
                coordslist = []
                distancelist = []
                timeslist = []
                kfspeed = new KalmanFilter
                kfdistance = new KalmanFilter

                addheader(textinput)
                id = navigator.geolocation.watchPosition(success, error, {
                    enableHighAccuracy: true,
                    maximumAge: 50,
                });
                iswatching = true
            } else {
                navigator.geolocation.clearWatch(id)
                iswatching = false
            }

        };
        
        // getting data on success storing for postprocessing
        const success = (location): void => {
            accuracy = location.coords.accuracy
            let lastlat;
            let lastlon;

            try {
                lastlat = coordslist[coordslist.length - 2][0]
                lastlon = coordslist[coordslist.length - 2][1]
            } catch (e) {
                lastlat = location.coords.latitude
                lastlat = location.coords.longitude
            }
            coordslist.push([location.coords.latitude, location.coords.longitude]);
            timeslist.push(location.timestamp)
            distancelist = distancelist.concat([
                getDistance(
                    location.coords.latitude,
                    location.coords.longitude,
                    lastlat,
                    lastlon,
                ),
            ]);
            fdistancelist = distancelist.concat([
                kfdistance.filter(getDistance(
                    location.coords.latitude,
                    location.coords.longitude,
                    lastlat,
                    lastlon,
                ),)
            ]);
            try {
                speed = distancelist[distancelist.length - 1] / (timeslist[timeslist.length - 1] / 1000)
                fspeed = kfspeed.filter(speed)
            } catch {
                speed = 0
                fspeed = kfspeed.filter(0)
            }
            let timetolast;
            try {
                timetolast = location.timestamp - timeslist[timeslist.length - 2]
            } catch (e) {
                timetolast = 0
            }
            const locationdata: { timetolast: any; distance: any; fspeed: any; accuracy: any; fdistance: any; lat: any; long: any; speed: any; timestamp: any } = {
                accuracy: location.coords.accuracy,
                distance: distancelist[distancelist.length - 1],
                fdistance: kfdistance.filter(distancelist[distancelist.length - 1]),
                lat: location.coords.latitude,
                long: location.coords.longitude,
                speed: speed,
                fspeed: fspeed,
                timestamp: location.timestamp,
                timetolast: timetolast
            }
            console.dir(locationdata)
            adddata(locationdata)
        };

        // simple error catching function for position
        const error = (error: any) => {
            distancelist = [error];
            console.log(error);
        };



    ///////////////////   data saving and organising   ///////////////////

        // add header to indexed db to identify different tracks
        function addheader(name: string) {
            const request = db.transaction(['data'], 'readwrite')
                .objectStore('data')
                .add({
                    accuracy: name,
                    distance: name,
                    fdistance: name,
                    lat: name,
                    long: name,
                    speed: name,
                    fspeed: name,
                    timestamp: name,
                    timetolast: name
                });
            request.onsuccess = event => {
                console.log("Added header to database");
            };
            request.onerror = event => {
                console.log("Error adding header to database");
            };
        }
        
        // saving data as new row to indexed db
        function adddata(data) {
            const request = db.transaction(['data'], 'readwrite')
                .objectStore('data')
                .add(data);
            request.onsuccess = event => {
                console.log("Added data to database");
            };
            request.onerror = event => {
                console.log("Error adding data to database");
            };
        }

        // converts indexed db data to csv file for later use
        function download() {
            const txn = db.transaction('data', 'readonly');
            const store = txn.objectStore('data');
            let values = [];

            let query = store.getAll();

            query.onsuccess = (event) => {
                if (!event.target.result) {
                    console.log(`eror`);
                } else {
                    console.dir(query.result);
                    values = query.result
                    let csvContent = "data:text/csv;charset=utf-8,";
                    csvContent += "accuracy m; distance m; fdistance m; lat °; long °; speed m/s; fspeed m/s; timestamp; timetolast ms\r\n";

                    values.forEach(function (rowmap) {
                        let row = `${rowmap.accuracy}; ${rowmap.distance}; ${rowmap.fdistance}; ${rowmap.lat}; ${rowmap.long}; ${rowmap.speed}; ${rowmap.fspeed}; ${rowmap.timestamp}; ${rowmap.timetolast}`;
                        csvContent += row + "\r\n";
                    });
                    var encodedUri = encodeURI(csvContent);
                    var link = document.createElement("a");
                    link.setAttribute("href", encodedUri);
                    link.setAttribute("download", "data.csv");
                    document.body.appendChild(link); // Required for FF
                    link.click();
                }
            };

            query.onerror = (event: { target: { errorCode: any; }; }) => {
                console.log(event.target.errorCode);
            }
        }

    ///////////////////   post processing and visualisation   ///////////////////

        // calculating the speed, only for quick checking
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

        // creates data to show leaflet map on phone for checking
        function showmap() {

            const txn = db.transaction('data', 'readonly');
            const store = txn.objectStore('data');
            let values = [];
            let coordsindex = -1

            let query = store.getAll();

            query.onsuccess = (event) => {
                if (!event.target.result) {
                    console.log(`error`);
                } else {
                    values = query.result
                    values.forEach(function (rowmap) {
                        if (typeof rowmap.lat !== "number" || typeof rowmap.long !== "number") {
                            coords.push([])
                            coordsindex += 1
                        } else {
                            coords[coordsindex].push([rowmap.lat, rowmap.long])
                        }
                    });
                }
                coords = coords
            };


        }
</script>

<main>
    <input bind:value={textinput} type="text" name="text" id="text">
    <button on:click={() => watchposition()}>{!iswatching ? "Watch position" : "Stop watching"}</button>
    <button on:click={() => showmap()}>Map</button>
    {#if !iswatching}
        <button on:click={() => download()}>Download</button>
    {/if}

    <pre><code>accuracy: {accuracy.toFixed(2)}</code></pre>
    <pre><code>geschwi.: {(speed).toFixed(2)}</code></pre>
    <button on:click={() => getpos()}>Test Accuracy</button>

    {#if coords.length > 0}
        <LeafletMap><LeafletPolyline latlongs={coords}/></LeafletMap>
    {/if}

</main>

