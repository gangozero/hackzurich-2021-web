<script>
    import Map from "@arcgis/core/Map";
    import MapView from "@arcgis/core/views/MapView";
    import esriConfig from "@arcgis/core/config";
    import FeatureLayer from "@arcgis/core/layers/FeatureLayer";
    import Graphic from "@arcgis/core/Graphic";
    import {solve} from "@arcgis/core/rest/route";
    import RouteParameters from "@arcgis/core/rest/support/RouteParameters";
    import FeatureSet from "@arcgis/core/rest/support/FeatureSet";


    import {onMount} from 'svelte';
   

    esriConfig.apiKey = "AAPK3ace7928316549b28c33b66ea457e676dbsV_3c1DDacVuVpeGLLORLCTguTWMzszmAq_TwOPq10LjIj__CNgFsd1GDFUUHa";

    const map = new Map({
        basemap: "arcgis-navigation" //Basemap layer service
    });

    const flBarrier = new FeatureLayer({
        portalItem: { // autocasts as esri/portal/PortalItem
          id: "ab54728e3c794d588a4ea1720db8d10e"
        }
    });

    map.add(flBarrier); // adds the layer to the map

    const routeUrl = "https://route-api.arcgis.com/arcgis/rest/services/World/Route/NAServer/Route_World";


    onMount(async ()=>{
        const result = await flBarrier.queryFeatures();
        const barriers = result.features;

        const view = new MapView({
            container: "viewDiv",
            map: map,
            center: [8.51631, 47.38935], //Longitude, latitude
            zoom: 14
        });

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
                returnDirections: true,
                polygonBarriers: barriers
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

<div id="viewDiv" style="height: 100vh;"></div>

<style>
    @import "https://js.arcgis.com/4.21/@arcgis/core/assets/esri/themes/dark/main.css";
</style>