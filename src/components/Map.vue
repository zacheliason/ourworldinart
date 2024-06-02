<template> 
    <div ref="mapContainer" class="map-container"></div>
    <div id='popup' v-if="current_popup !== ''">
        <div @click='() => current_popup=""' class="x"></div>
        <div id='pop' v-html="current_popup"></div>
    </div>
    <div id="filters">
        <div id="filter-box" :class="{ 'rounded-right-corner': !show_tab }">
          <label></label>
          <!-- <label>Filter by artist: </label> -->
          <input style='width:calc(100% - 1em)' type="text" list='artistList' v-model="artistSearchQuery" placeholder="Filter by artist...">
          <datalist id="artistList">
            <option v-for="artist in filteredArtists" :key="artist" :value="artist">{{ artist }}</option>
          </datalist>
        </div>
        <div id="filter-tab" v-if="show_tab" @click="() => show_tab=false" title="click to minimize">
            <p>Click on individual artpieces (the white circles) to pull up metadata associated with each one. The artworks on the map continuously filter as you type inside the artists box.</p>
            <p>This map contains all pieces of art I could (easily) find on English Wikipedia and Wikiarts. I made it because I kept wishing I knew where to find paintings by Marc Chagall.</p>
            <p>Needless to say this map isn't comprehensize and should be fact checked!</p>
            <p>The color theme for this website is <a target="_blank" href='https://en.wikipedia.org/wiki/International_orange'>International Orange</a>; the type is set in <span class='key'><a target="_blank" href='https://en.wikipedia.org/wiki/Futura_(typeface)'>Futura</a></span> and <span style="font-style:italic"><a target="_blank" href='https://en.wikipedia.org/wiki/Mrs_Eaves'>Mrs Eaves</a></span>.</p>
            <p v-if="out_of_utah" title="">
                <hr>
                <p>The costs of hosting this website are low (but not zero) and I am a student! Tip if you like.</p>
                <div class="stripe-center">
                    <stripe-buy-button
                    buy-button-id="buy_btn_1PMmbDKc9EdKPsttWXvRCHhy"
                    publishable-key="pk_live_51PMmGsKc9EdKPsttzQaandDD1cUFPhAe5D9UeR0iT5gyXOs6gxMZUDF5J7T5XMivMnswRVqxTQGdP9z4PqtHYLy100zZlYNnzm"
                    >
                    </stripe-buy-button>
                </div>
            </p>
            
            <div class="icons">
                <a href="https://github.com/zacheliason" title="Zach Eliason Github"><i class="fa-brands fa-github"></i></a>
                <a href="https://github.com/zacheliason/ourworldinart" title="Source code"><i class="fa fa-code"></i></a>
                <a href="https://zacheliason.com" title="My personal website"><i class="fa fa-user"></i></a>
            </div>
        </div>
        <div v-else @click="() => show_tab=true" id="show_filter_tab" class='chevron-tab'>
            <div class='chevron'></div>
        </div>
    </div>
</template>

<script>
// import { ref } from 'vue'
import mapboxgl from 'mapbox-gl';
mapboxgl.accessToken = "pk.eyJ1IjoiemNoZSIsImEiOiJjbHdpMGo5ZG4wazBiMmlxcmpoM2ZkODB0In0.5QZCNahVxBS40uHJu7ONww";

export default {
    data() {
      const mobile = this.isMobile()
      return {
        isMobile: mobile,
        artists: [],
        artistsMap: null,
        artworks: null,
        artistSearchQuery: '',
        map: null,
        current_popup: '',
        out_of_utah: false,
        previousActiveMarker: null,
        show_tab: !mobile
      };
    },

    computed: {
      filteredArtists() {
        return this.artists.filter(artist => {
            return artist.toLowerCase().includes(this.artistSearchQuery.toLowerCase());
        }).sort();
        },

        filteredArtworks() {
            if (!this.artworks) {
                return [];
            }

            const searchQuery = this.artistSearchQuery.trim().toLowerCase();

            if (searchQuery === '') {
                return this.to_geojson(this.jitter(this.join_location(this.artworks, this.locations)))
            }

            let artworks = this.artworks.filter((artwork) => {
                return artwork.artist.toLowerCase().includes(searchQuery);
            });

            artworks = this.join_location(artworks, this.locations)
            artworks = this.jitter(artworks)

            return this.to_geojson(artworks)
        },
    },

    mounted() {
        if(navigator.userAgent.indexOf('iPhone') > -1 ) {
            document
            .querySelector("[name=viewport]")
            .setAttribute("content","width=device-width, initial-scale=1, maximum-scale=1");
        }

        let ip = ""

        fetch('https://api.ipify.org?format=json')
        .then(x => x.json())
        .then(({ ip }) => {
            this.ip = ip;
        });

        const getIPInfo = async () => {
            const url = `http://ip-api.com/json/${ip}`;

            try {
                const response = await fetch(url);
                
                if (!response.ok) {
                    throw new Error('Network response was not ok ' + response.statusText);
                }

                const data = await response.json();
                if ((('country' in data) && data.country != "United States") || ('country' in data && data.country == "United States" && 'region' in data && data.region != "UT")) {
                    // comment back in for payment tab
                    this.out_of_utah = false;
                }
            } catch (error) {
                // console.log(error.message)
            }
        };

        getIPInfo()

        fetch('artists_wikipedia.json')
            .then(response => response.json())
            .then(jsonData => {
                this.artistsMap = jsonData
                this.artists = Object.keys(jsonData); 
            })
            .catch(error => {
                console.error(error);
            });
 

        fetch('locations.json')
            .then(response => response.json())
            .then(jsonData => {
                this.locations = jsonData
            })
            .catch(error => {
                console.error(error);
            });
 

        fetch('artworks.json')
            .then(response => response.json())
            .then(jsonData => {
                this.artworks = jsonData
                this.initMap(this.to_geojson(this.jitter(this.join_location(this.artworks, this.locations))));
            })
            .catch(error => {
                console.error(error);
            });
    },
    methods: {
        isMobile() {
            return (/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)) 
        }, 

        // This function finds overlapping points and clusters them in optimized concentric circles about the origin
        jitter(points) {
            const pointsMap = new Map();
            for (const point of points) {
                const key = `${point.Latitude},${point.Longitude}`;
                if (!pointsMap.has(key)) {
                    pointsMap.set(key, []);
                }
                pointsMap.get(key).push(point);
            }

            for (const [key, group] of pointsMap.entries()) {
                if (group.length > 1) {
                    const [latitude, longitude] = key.split(",").map(Number);
                    const packedCoords = this.packPoints(group.length, latitude, longitude);

                    for (let i = 0; i < group.length; i++) {
                        const point = group[i];
                        const [x, y] = packedCoords[i];
                        point.Latitude = x;
                        point.Longitude = y;
                    }
                }
            }

            return points;
        },

        // helper function for jitter
        packPoints(numPoints, xStart, yStart) {
            const packedPoints = [];
            let subdivisions = [1];
            let numRings = 1;
            let x = xStart;
            let y = yStart;
            let r = 1;

            // Calculate the number of points that should be in the first circle
            let pointsTotalInCircle = Math.floor(2 * Math.PI);

            let rt = 1 + pointsTotalInCircle;
            const ringSizes = [pointsTotalInCircle];
            const totalCountsWithRing = [rt];
            while (rt < numPoints) {
                r += 1;
                pointsTotalInCircle = Math.floor(2 * Math.PI * r);
                ringSizes.push(pointsTotalInCircle);
                rt += ringSizes[ringSizes.length - 1];
                totalCountsWithRing.push(rt);
            }

            if (totalCountsWithRing.length > 1) {
                ringSizes.pop();
                totalCountsWithRing.pop();

                rt = totalCountsWithRing[totalCountsWithRing.length - 1];
                numRings = ringSizes.length;
                const diff = numPoints - rt;

                r = 1.5; // This can be adjusted based on the required exponential growth rate

                for (let i = 1; i < numRings; i++) {
                    subdivisions.push(subdivisions[subdivisions.length - 1] * r);
                }

                const total = subdivisions.reduce((sum, x) => sum + x, 0);
                    subdivisions.forEach((x, i, arr) => {
                    arr[i] = Math.round((x * diff) / total);
                });

                const currentSum = subdivisions.reduce((sum, x) => sum + x, 0);

                if (currentSum !== diff) {
                    const difference = diff - currentSum;
                    subdivisions[subdivisions.length - 1] += difference;
                }
            } else {
                subdivisions = [numPoints - totalCountsWithRing[0]];
            }

            r = 1;
            pointsTotalInCircle = Math.floor(2 * Math.PI * r);

            if (subdivisions.length > 0) {
                pointsTotalInCircle += subdivisions[r - 1];
            }

            // scales the transformation down appropriately
            const scalingFactor = 0.0002;
            // corrects the skew in shape from applying a circular pattern to a sphere
            const curvatureCorrection = Math.cos((xStart * Math.PI) / 180);

            packedPoints.push([x, y]);

            for (let i = 0; i < numRings; i++) {
                const angles = Array.from({ length: pointsTotalInCircle }, (_, j) => (j * 2 * Math.PI) / pointsTotalInCircle);
                const xs = angles.map((angle) => (scalingFactor * curvatureCorrection * r * Math.cos(angle)) + xStart);
                const ys = angles.map((angle) => (scalingFactor * r * Math.sin(angle)) + yStart);

                packedPoints.push(...xs.map((x, j) => [x, ys[j]]));

                if (packedPoints.length < numPoints) {
                    r += 1;
                    pointsTotalInCircle = Math.floor(2 * Math.PI * r);
                    pointsTotalInCircle += subdivisions[r - 1];
                }
            }

            if (packedPoints.length !== numPoints) {
                throw new Error(`Expected ${numPoints} points, but got ${packedPoints.length} points`);
            }

            return packedPoints;
        },

        // convert artworks to geojson format
        to_geojson(artworks){
            let ctr = 0
            const features = artworks.map(artwork => {
            const { Latitude, Longitude, location_id, ...artworkProps } = artwork;

            return {
                type: 'Feature',
                geometry: {
                    type: 'Point',
                    coordinates: [Longitude, Latitude]
                },
                properties: {
                    ...artworkProps
                },
                id: ctr++
            };
            });

            return {
            type: 'FeatureCollection',
                features
            };
        },

        // join locations and artworks datasets
        join_location(artworks, locations) {
            return artworks.map(artwork => {
                const location = locations[artwork.location_id];
                return {
                    ...artwork,
                    Latitude: location[0],
                    Longitude: location[1],
                    location: location[2],
                    location_header: location[3]
                };
            });
        },

        // set up map
        initMap(geojson) {
            const map = new mapboxgl.Map({
                container: this.$refs.mapContainer,

                // darkmode
                style: "mapbox://styles/mapbox/dark-v11?optimize=true",
                // style: "mapbox://styles/mapbox/streets-v12"

                // Different starting points for mobile / desktop
                center: this.isMobile ? [-73.9639614009125, 40.78248611767528] : [2.326929220445182, 48.86008251391034],
                zoom: this.isMobile ? 14 : 15.6,
            });

            map.on('load', () => {
                map.addSource('markers', {
                    type: 'geojson',
                    data: geojson,
                    cluster: true,
                    clusterMaxZoom: 14,
                    clusterRadius: 50
                });

                map.addLayer({
                    id: 'clusters',
                    type: 'circle',
                    source: 'markers',
                    filter: ['has', 'point_count'],
                    paint: {
                        'circle-color': [
                            'interpolate',
                            ['linear'],
                            ['get', 'point_count'],
                            0, '#fffa6b',
                            2000, '#FF4F00' 
                        ],
                        'circle-radius': [
                            'interpolate',
                            ['linear'],
                            ['get', 'point_count'],
                            2, 15,
                            100, 25,
                            750, 26,
                            1000, 27,
                            1500, 28,
                            2000, 30
                        ],
                    }
                });

                map.addLayer({
                    id: 'cluster-count',
                    type: 'symbol',
                    source: 'markers',
                    filter: ['has', 'point_count'],
                    layout: {
                        'text-field': '{point_count_abbreviated}',
                        'text-font': ['DIN Offc Pro Medium', 'Arial Unicode MS Bold'],
                        'text-size': 12
                    }
                });

                map.addLayer({
                    id: 'unclustered-point',
                    type: 'circle',
                    source: 'markers',
                    filter: ['!', ['has', 'point_count']], 
                    paint: {
                        'circle-color': [
                            'case',
                            ['boolean', ['feature-state', 'active'], false],
                            '#FF4F00', 
                            'white' // Color for inactive markers
                            ],
                        'circle-radius': 6,
                    }
                }, 'clusters');
            });


            map.on('click', 'unclustered-point', (e) => {
                const feature = e.features[0];


                const setPopup = async () => {
                    const delay = ms => new Promise(res => setTimeout(res, ms));
                    if (this.isMobile) {
                        await delay(1000)
                    }
                    this.current_popup = this.generatePopupHTML(feature.properties)
                }

                setPopup()

                const coordinates = e.features[0].geometry.coordinates;
                    
                map.easeTo({
                    center: coordinates, 
                    zoom: 16, 
                    duration: 1000 
                });

                const sourceId = feature.source;
                const featureId = feature.id;

                map.setFeatureState(
                    { source: sourceId, id: featureId },
                    { active: true }
                );

                if (this.previousActiveMarker !== null && this.previousActiveMarker.id != featureId) {
                    map.removeFeatureState(this.previousActiveMarker, 'active');
                }

                this.previousActiveMarker = { source: sourceId, id: featureId };
            });

            map.on('click', 'clusters', function (e) {
                var features = e.features;
                var clusterId = features[0].properties.cluster_id;
                map.getSource('markers').getClusterExpansionZoom(clusterId, function (err, zoom) {
                    if (err)
                        return;

                    map.easeTo({
                        center: features[0].geometry.coordinates,
                        zoom: zoom
                    });
                });
            });

            map.on('mouseenter', 'unclustered-point', function () {
                map.getCanvas().style.cursor = 'pointer';
            });

            map.on('mouseleave', 'unclustered-point', function () {
                map.getCanvas().style.cursor = '';
            });


            map.on('mouseenter', 'clusters', function () {
                map.getCanvas().style.cursor = 'pointer';
            });

            map.on('mouseleave', 'clusters', function () {
                map.getCanvas().style.cursor = '';
            });

            this.map = map;
        },

        generatePopupHTML(artwork) {
            let html = "<div style='width:100%;'><div class='m-spacer'></div><div class='m-spacer'></div>"

            if ('image' in artwork) {
                html += `<div style='text-align:center;'><img class='artwork-image' src=${artwork.image}></div>`
            } else{
                html += `<div class='spacer'></div>`
            }

            html += `<h2 class="artwork-title">${artwork.title}</h2>`

            if ('location_header' in artwork) {
                html += `<div class='artwork-location'>`
                html += `${artwork.location_header}</div>`
            }

            html += "<hr>"
            const skipKeys = ['title', 'image', 'location_header', 'use_date']

            // Define the order of keys
            const keyOrder = ['artist', 'location', 'Category', 'date', ...Object.keys(artwork).filter(key => (!skipKeys.includes(key)) && !['artist', 'date', 'Category', 'location'].includes(key)).sort()];

            for (const key of keyOrder) {
                if (skipKeys.includes(key) || !artwork.hasOwnProperty(key)) {
                    continue;
                }
                const value = artwork[key];
                html += `<div class='value'><span class='key'>${key}:</span>${value}</div><br>`;
            }

            const use_date = artwork.use_date
            if (use_date > -1) {
                html += `<div class='value'><span class='key'>artwork source: </span>`
                let testTitle = artwork.title.toLowerCase().replace(/[().,/]/g, '').replace(/[\s:']+/g, '-').trim();
                let testArtist = artwork.artist.normalize('NFD').replace(/[\u0300-\u036f]/g, '').toLowerCase().replace(/[().,/]/g, '').replace(/[\s:']+/g, '-').trim();

                html += `<a target='_blank' class='link-value' href="https://www.wikiart.org/en/${testArtist}/${testTitle}`
                if (use_date == 1){
                    html += `-${artwork.date}">`
                } else {
                    html += `">`
                }
                html += "wikiart.org</a></div>"
            } else if (use_date == -2) {

                const alt_url_artists = ['Ivan Albright', 'John Middleton', 'Louise Bourgeois', 'Marc Chagall', 'Picasso']
                if (artwork.artist in alt_url_artists) {
                    html += `<div class='value'><span class='key'>artwork source: </span><a class='link-value' target='_blank' href="https://en.wikipedia.org/wiki/List_of_artworks_by_${artwork.artist.replaceAll(' ', '_')}">list of works by ${artwork.artist.toLowerCase()}</a></div>`
                } else {
                    html += `<div class='value'><span class='key'>artwork source: </span><a class='link-value' target='_blank' href="https://en.wikipedia.org/wiki/List_of_works_by_${artwork.artist.replaceAll(' ', '_')}">list of works by ${artwork.artist.toLowerCase()}</a></div>`
                }
            }

            if (this.artistsMap[artwork.artist]) {
                html += `<div class='value'><span class='key'><i class="fa-brands fa-wikipedia-w fa-lg"></i>: </span><a class='link-value' target='_blank' href="${this.artistsMap[artwork.artist]}">${artwork.artist.toLowerCase()} wikipedia page</a></div>`
            }

            html += "<div class='spacer'></div><div class='spacer'></div></div>"

        return html; 
      },
    },

    watch: {
        filteredArtworks(geojson) {
            if (this.map) {
                try{
                    this.map.getSource('markers').setData(
                        geojson
                    );
                } catch(e) {
                    // console.log(e)
                }
            } else {
                this.initMap(geojson);
            }
        },
    },
    unmounted() {
        this.map.remove();
        this.map = null;
    }
};

</script>

<style>

body {
    padding: 0;
    margin: 0;
}

img {
    background-color: #f3f3f3;
    width: 100%;
}

.map-container {
  flex: 1;
}

.stripe-center {
    display:flex;
    justify-content: center;
}

#filters label {
    font-family: 'futura-pt', 'sans-serif';
    font-weight: 600 !important;
}


#filters input {
    font-family: 'futura-pt', 'sans-serif';
    font-weight: 600;
    /* text-transform: capitalize; */
}

#filters {
    display: inline-block;
    position: fixed;
    top: 50px;
    left: 50px;
    color:white;
    min-width: 300px;
}

#filter-box {
    font-weight: 600;
    text-transform: uppercase;
    background-color: rgba(85,85,85,.9);
    border-radius: 5px 5px 0 0;
    padding:1em 2.5em;
}

.rounded-right-corner {
    border-radius: 5px 5px 5px 0 !important;
}

#filter-tab {
    cursor: pointer;
    max-width: 310px;
    overflow-y: scroll;
    font-size: 1.2em;
    font-family: mrs-eaves, 'serif';
    text-transform: unset;
    background-color: rgba(255,255,255,1);
    border-radius: 0 0 5px 5px;
    color: black;
    padding:1em 2em;
    max-height: calc(100vh - 203.5px)
}

.icons {
    display: flex;
    justify-content: space-between;
    font-size: 1.5em;
}

.icons a {
    padding: 0 .3em;
}

#popup {
    /* display:flex; */
    align-items: center;
    position: fixed;
    right:0;
    top:0;
    height:100vh;
    background-color: white;
    width:20vw;
    overflow-y: scroll;
    padding: 1em 1.5em 1.8em 1.5em;
    transition:.6s;
}

.x:hover {
    cursor: pointer;
    background-color: #f3f3f3;
    transition:.3s;
}

.x {
    position: relative;
    width: 40px;
    height: 40px; 
    display: inline-block;
    float: right;
    margin-bottom: 1em;
    z-index: 9999;
    cursor: pointer;
}

.x:hover .x::before, .x::after {
    background-color: black;
}

.x::before,
.x::after {
    content: '';
    position: absolute;
    top: 50%;
    left: 50%;
    width: 60%;  /* Length of the lines */
    height: 3px; /* Thickness of the lines */
    background-color: #FF4F00; /* Color of the lines */
}

.x::before {
    transform: translate(-50%, -50%) rotate(45deg);
}

.x::after {
    transform: translate(-50%, -50%) rotate(-45deg);
}


input {
    background-color: rgba(200,200,200,.5);
    color: white;
    border: 0;
    /* border: 2px solid #FF4F00; */
    border-radius: 0;
    padding: .5em .5em .4em .5em;
    font-size: 16px;
}

.artwork-title {
    font-family: 'futura-pt', serif;
    font-weight: 600;
    text-align:center;
    color:#FF4F00;
    margin: 0;
    padding-bottom: 0;
    text-transform: uppercase;
}

.artwork-location {
    font-weight: 600;
    text-align: center;
    font-family: "mrs-eaves-roman-petite-caps", serif;
    text-transform: uppercase;
    opacity: .6;
}

.key {
    font-style:normal;
    font-family: 'futura-pt', serif;
    font-weight: 600;
    display: inline-block;
    text-transform: uppercase;
    padding-right: .3em;
}



.link-value {
    color: #FF4F00;
    text-decoration: underline;
    font-weight:600;
    font-family: mrs-eaves-roman-small-caps, serif;
    text-transform: unset;
    padding-bottom: .3em;
}

.value {
    display: inline-block;
    font-family: mrs-eaves, serif;
    text-transform: unset;
    padding-bottom: .3em;
    width: 100%;
}

hr {
margin-bottom: 1.3em;
}

.mapboxgl-ctrl-logo, .mapboxgl-ctrl-attrib {
display: none !important;
}

.artwork-image {
    margin-bottom: 1em;
}

.chevron-tab {
    position: relative;
    width: 2.5em;
    height: 30px; /* Adjust height as needed */
    background-color: white;
    cursor: pointer;
    border-radius: 0 0 5px 5px;
    opacity: .9;
}

.chevron-tab {
    opacity: 1;
}

.chevron {
    position: absolute;
    top: 40%;
    left: 50%;
    width: 10px;
    height: 10px;
    transform: translate(-50%, -50%) rotate(45deg);
    border-right: 3px solid #FF4F00;
    border-bottom: 3px solid #FF4F00;
}

.futura {
    font-family: futura !important;
}

button.BuyButton-ButtonText {
    font-family: futura !important;
    text-transform: uppercase !important;
    width:100%;
}

.BuyButton-ButtonTextContainer.BuyButton-ButtonText, button {
    font-family: futura !important;
    text-transform: uppercase !important;
    width:100%;
}

body, html {
            margin: 0;
            padding: 0;
            height: 100%;
            overflow: hidden; /* Prevents the body from scrolling */
        }

.spacer {
    height: 100px;
}


@media (max-width: 600px) {
    .x {
        position: fixed;
        top: 1.5em;
        right: 1.5em;
        background-color: white;
    }

    img {
        margin-top: 10vh;
    }

    #popup {
        display: flex;
        position: absolute;
        left: 0;
        bottom: 0;
        background-color: white;
        width: calc(100vw - 3em);
        height: 100vh;
        z-index: 99999 !important;
        padding: 1.5em;
    }

    #pop {
        position:absolute;
        top: 1.5em;
        overflow-y: scroll;
        max-width:100%;
        width: calc(100vw - 3em);
    }

    #filters {
        display: inline-block;
        position: fixed;
        top: 0;
        left: 0;
        color:white;
        width: 100vw;
        padding-right:0;
    }

    #filter-box {
        background-color: grey;
        border-radius: 0 !important;
    }

    input {
        border: 2px solid white;
    }

    input[type='text'],
    input[type='number'],
    textarea {
        font-size: 16px;
    }

    #filter-tab {
        max-width: unset;
        max-height: unset;
        padding:0 1em 1em;
    }
    #filters input {
        margin: 0;
        width: calc(100vw - 4em);
        border: 0;
        border-radius: 0;
        background-color: rgba(200,200,200,1);
        color: white;
    }

    #filters input::placeholder {
        color: white;
        opacity:.7;
    }
}



</style>