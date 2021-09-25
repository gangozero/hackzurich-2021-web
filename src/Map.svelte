<script>
    import Map from "@arcgis/core/Map";
    import MapView from "@arcgis/core/views/MapView";
    import esriConfig from "@arcgis/core/config";
    import FeatureLayer from "@arcgis/core/layers/FeatureLayer";
    import GraphicsLayer from "@arcgis/core/layers/GraphicsLayer";
    import Graphic from "@arcgis/core/Graphic";
    import {solve} from "@arcgis/core/rest/route";
    import RouteParameters from "@arcgis/core/rest/support/RouteParameters";
    import FeatureSet from "@arcgis/core/rest/support/FeatureSet";
    import {fetchServiceDescription} from "@arcgis/core/rest/networkService";

    import { addFeatures } from '@esri/arcgis-rest-feature-layer';
    import Point from "@arcgis/core/geometry/Point";



    import {onMount} from 'svelte';
   

    esriConfig.apiKey = "AAPK3ace7928316549b28c33b66ea457e676dbsV_3c1DDacVuVpeGLLORLCTguTWMzszmAq_TwOPq10LjIj__CNgFsd1GDFUUHa";

    let userPosition = [8.51631, 47.38935];
    let loading = true;

    const simpleFillUGC = {
        type: "simple-fill",
        color: [227, 139, 79, 0.8],  // Orange, opacity 80%
        outline: {
            color: [255, 255, 255],
            width: 1
        }
    };

    const simpleFillForecast = {
        type: "simple-fill",
        color: [240, 71, 255, 0.4],  // Orange, opacity 80%
        outline: {
            color: [255, 255, 255],
            width: 1
        }
    };
    

    const map = new Map({
        basemap: "arcgis-navigation" //Basemap layer service
    });

    const flBarrier = new FeatureLayer({
        portalItem: { // autocasts as esri/portal/PortalItem
          id: "ab54728e3c794d588a4ea1720db8d10e"
        }
    });

    const flUGC = new FeatureLayer({
        portalItem: {
          id: "bccc1003a4f44fa2ba488697efb73564"
        },
        renderer: {
            type: "simple",
            symbol: simpleFillUGC
        }
    });

    const flForecast = new FeatureLayer({
        portalItem: {
          id: "6a4e5e0f15984191826b982a737c2e9c"
        },
        renderer: {
            type: "simple",
            symbol: simpleFillForecast
        }
    });

    const flShelters = new FeatureLayer({
        portalItem: { 
          id: "f0f922a956b046ae88fadc8b32e1fb7f"
        },
        renderer: {
            type: "simple",
            "symbol": {
                "type": "picture-marker",
                "url": "http://static.arcgis.com/images/Symbols/NPS/npsPictograph_0231b.png",
                "width": "18px",
                "height": "18px"
            }
        }
    });

    const graphicsLayer = new GraphicsLayer();
    
    map.add(graphicsLayer);
    map.add(flForecast); 
    map.add(flBarrier); 
    map.add(flUGC); 
    map.add(flShelters); 


    async function getSheltersCoords(){
        const {features} = await flShelters.queryFeatures();
        return features.map(f=>[f.geometry.longitude,f.geometry.latitude]);
    }

    function getPosition(){
        return new Promise(resolve =>{
            navigator.geolocation.getCurrentPosition(pos =>{
                resolve([pos.coords.longitude,pos.coords.latitude]);
            })
        });
    }
    
    

    const routeUrl = "https://route-api.arcgis.com/arcgis/rest/services/World/Route/NAServer/Route_World";
  
    onMount(async ()=>{
        if(window.location.hash == '#gps'){
            userPosition = await getPosition();
        }
        

        const serviceDescription = await fetchServiceDescription(routeUrl);
        const { supportedTravelModes } = serviceDescription;
        const travelMode = supportedTravelModes.find((mode) => mode.name === "Walking Time");

        const barriers = await flBarrier.queryFeatures();
        const UGCs = await flUGC.queryFeatures();
        const sheltersCoords = await getSheltersCoords();

        loading=false;

        const view = new MapView({
            container: "viewDiv",
            map: map,
            center: userPosition, //Longitude, latitude
            zoom: 14
        });
          
        let pointGraphic = new Graphic({
            geometry: {
                type: "point", 
                longitude: userPosition[0],
                latitude: userPosition[1]
            },
            symbol: {
                type: "simple-marker",  
                color: [226, 119, 40]
            }
        });

        view.graphics.add(pointGraphic)

        

        view.on("click",async (event) => {

            const y1 = event.mapPoint.latitude-0.00005;
            const y2 = event.mapPoint.latitude+0.00005;
            const x1 = event.mapPoint.longitude-0.000075;
            const x2 = event.mapPoint.longitude+0.000075;

            const rings = [[x1,y1],[x2,y1],[x2,y2],[x1,y2],[x1,y1]];

            const result = await addFeatures({
                url: 'https://services8.arcgis.com/VAHmVwXyn8lMPYKG/arcgis/rest/services/flood_ugc2/FeatureServer/0',
                features: [{
                    attributes: {
                        type: 'UGC_FLOOD'
                    },
                    geometry: {
                        "rings": [rings],
                        "spatialReference": {
                            "wkid": 4326
                        }
                    }
                }]
            });


            const polygon = {
                type: "polygon",
                rings: rings
            };

            

            const polygonGraphic = new Graphic({
                geometry: polygon,
                symbol: simpleFillUGC,
            });

            graphicsLayer.add(polygonGraphic);
        })

        async function getRoute(from,to,main) {
          
            const routeParams = new RouteParameters({
                stops: new FeatureSet({
                    features: [
                        new Graphic({
                        geometry: new Point({
                            latitude: from[1],
                            longitude: from[0]
                        })
                        }),
                        new Graphic({
                        geometry: new Point({
                            latitude: to[1],
                            longitude: to[0]
                        })
                        })
                    ]
                }),
                polygonBarriers: new FeatureSet({
                    features: [...barriers.features, ...UGCs.features]
                }),
                travelMode
            });
            const data = await solve(routeUrl, routeParams);

            data.routeResults.forEach(function(result) {
                result.route.symbol = {
                    type: "simple-line",
                    color: main ? [0, 153, 255, 0.7] : [150, 150, 150, 0.7],
                    width: 4
                };

                view.graphics.add(result.route);
            });
                
        }

        async function getRoutesToShelters(from){
            const closest = sheltersCoords.sort((crd1,crd2) =>{
                return Math.hypot(crd1[0]-from[0],crd1[1]-from[1])-Math.hypot(crd2[0]-from[0],crd2[1]-from[1]);
            })

            await getRoute(from, closest[2], false);
            await getRoute(from, closest[1], false);
            getRoute(from, closest[0], true);
        }


        getRoutesToShelters(userPosition);
    });
</script>

{#if loading}
<div class="loader">Loading...</div>
{/if}
<div id="viewDiv"></div>



<style>
    @import "https://js.arcgis.com/4.21/@arcgis/core/assets/esri/themes/dark/main.css";
    #viewDiv{
        height:100%;
        width: 100%;
        user-select: none;
    }

    .loader,
    .loader:after {
    border-radius: 50%;
    width: 10em;
    height: 10em;
    }
    .loader {
    position:absolute;
    top: 50%;
    margin: 60px auto;
    font-size: 10px;
    position: relative;
    text-indent: -9999em;
    border-top: 1.1em solid rgba(1, 144, 125, 0.2);
    border-right: 1.1em solid rgba(1, 144, 125, 0.2);
    border-bottom: 1.1em solid rgba(1, 144, 125, 0.2);
    border-left: 1.1em solid #ffffff;
    -webkit-transform: translateZ(0);
    -ms-transform: translateZ(0);
    transform: translateZ(0);
    -webkit-animation: load8 1.1s infinite linear;
    animation: load8 1.1s infinite linear;
       }
    

    @keyframes load8 {
    0% {
        -webkit-transform: rotate(0deg);
        transform: rotate(0deg);
    }
    100% {
        -webkit-transform: rotate(360deg);
        transform: rotate(360deg);
    }
    }

</style>