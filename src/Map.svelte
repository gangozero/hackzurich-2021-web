<script>
    import Map from "@arcgis/core/Map";
    import MapView from "@arcgis/core/views/MapView";
    import esriConfig from "@arcgis/core/config";

    import {onMount} from 'svelte';
    console.log(Map);

    esriConfig.apiKey = "AAPK3ace7928316549b28c33b66ea457e676dbsV_3c1DDacVuVpeGLLORLCTguTWMzszmAq_TwOPq10LjIj__CNgFsd1GDFUUHa";

    onMount(()=>{
        const map = new Map({
          basemap: "arcgis-topographic" // Basemap layer service
    });

    const view = new MapView({
          map: map,
          center: [33.981487, 62.267936], // Longitude, latitude
          zoom: 13, // Zoom level
          container: "viewDiv" // Div element
        });
    })
    
</script>

<div id="viewDiv" style="height: 100vh;"></div>

<style>
    @import "https://js.arcgis.com/4.21/@arcgis/core/assets/esri/themes/dark/main.css";
</style>