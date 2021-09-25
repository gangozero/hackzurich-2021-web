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

    const simpleFillUGC = {
                type: "simple-fill",
                color: [227, 139, 79, 0.8],  // Orange, opacity 80%
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

    const graphicsLayer = new GraphicsLayer();
    
    map.add(graphicsLayer);
    map.add(flBarrier); 
    map.add(flUGC); 

    const routeUrl = "https://route-api.arcgis.com/arcgis/rest/services/World/Route/NAServer/Route_World";


    onMount(async ()=>{
        const serviceDescription = await fetchServiceDescription(routeUrl);
        const { supportedTravelModes } = serviceDescription;
        const travelMode = supportedTravelModes.find((mode) => mode.name === "Walking Time");

        const barriers = await flBarrier.queryFeatures();

        const view = new MapView({
            container: "viewDiv",
            map: map,
            center: [8.51631, 47.38935], //Longitude, latitude
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

            console.log(rings)

        })

        view.on("click", function(event) {
            if (view.graphics.length === 0) {
                addGraphic("origin", event.mapPoint);
            } else if (view.graphics.length === 1) {
                addGraphic("destination", event.mapPoint);
                getRoute(); // Call the route service
            } else {
                view.graphics.removeAll();
                addGraphic("origin", event.mapPoint);
            }
        });

        function addGraphic(type, point) {
            const graphic = new Graphic({
            symbol: {
                type: "simple-marker",
                color: (type === "origin") ? "white" : "black",
                size: "8px"
            },
            geometry: point
            });
            view.graphics.add(graphic);
        }

        async function getRoute() {
            console.log(barriers);
            const routeParams = new RouteParameters({
                stops: new FeatureSet({
                    features: view.graphics.toArray()
                }),
                polygonBarriers: barriers,
                travelMode
            });
            const data = await solve(routeUrl, routeParams);

            data.routeResults.forEach(function(result) {
                result.route.symbol = {
                    type: "simple-line",
                    color: [5, 150, 255],
                    width: 3
                };

                view.graphics.add(result.route);
            });
                
        }
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