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


    const simpleFillUGC = {
                type: "simple-fill",
                color: [227, 139, 79, 0.8],  // Orange, opacity 80%
                outline: {
                    color: [255, 255, 255],
                    width: 1
                }
            };
    
    const simpleFillShelters = {
        type: "simple-fill",
        color: [255, 0, 0, 0.8],  // Red, opacity 80%
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
          id: "81d73c02a70d4aae83fc7c3ba0f073a7"
        },
        renderer: {
            type: "simple",
            symbol: simpleFillUGC
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
        const shelters = await getSheltersCoords();

        const view = new MapView({
            container: "viewDiv",
            map: map,
            center: userPosition, //Longitude, latitude
            zoom: 14
        });

        view.on("hold",async (event) => {

            const y1 = event.mapPoint.latitude-0.00005;
            const y2 = event.mapPoint.latitude+0.00005;
            const x1 = event.mapPoint.longitude-0.00005;
            const x2 = event.mapPoint.longitude+0.00005;

            const rings = [[x1,y1],[x2,y1],[x2,y2],[x1,y2],[x1,y1]];

            const result = await addFeatures({
                url: 'https://services8.arcgis.com/VAHmVwXyn8lMPYKG/arcgis/rest/services/flood_ugc/FeatureServer/0',
                features: [{
                    attributes: {
                        type: 'local flood'
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

        async function getRoute(from,to,width) {
            console.log(barriers);
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
                polygonBarriers: barriers,
                travelMode
            });
            const data = await solve(routeUrl, routeParams);

            data.routeResults.forEach(function(result) {
                result.route.symbol = {
                    type: "simple-line",
                    color: [5, 150, 255],
                    width: width || 3
                };

                view.graphics.add(result.route);
            });
                
        }

        async function getRoutesToShelters(from){
            const closest = shelters.sort((crd1,crd2) =>{
                return Math.hypot(crd1[0]-from[0],crd1[1]-from[1])-Math.hypot(crd2[0]-from[0],crd2[1]-from[1]);
            })

            getRoute(from, closest[0], 4);
            getRoute(from, closest[1], 3);
            getRoute(from, closest[2], 2);
        }


        getRoutesToShelters(userPosition);
    });
</script>

<div id="viewDiv"></div>

<style>
    @import "https://js.arcgis.com/4.21/@arcgis/core/assets/esri/themes/dark/main.css";
    div{
        height:100%;
        width: 100%;
    }
</style>