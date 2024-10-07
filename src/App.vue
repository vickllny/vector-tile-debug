<template>
  <div>
    <div id="map"></div>
  </div>
</template>

<script>

import { ref } from 'vue';

import { VectorTile as VectorTileSource } from 'ol/source';
import VectorTile from 'ol/layer/VectorTile';
import MVT from 'ol/format/MVT';
import { WMTS } from 'ol/tilegrid';
import { Projection } from 'ol/proj';
import { View } from 'ol';
import { Map } from 'ol';
import { defaults as defaultControls } from 'ol/control/defaults';
import { TileDebug } from 'ol/source';
import TileLayer from 'ol/layer/Tile';

import { Fill, Stroke, Style, Text } from 'ol/style';


const map = ref(null)

export default {
  data() {
    return {
      baseUrl: "/api/geoserver/gwc/service/wmts",
      gridsetName: "EPSG:4326",
      gridNames: ['EPSG:4326:0', 'EPSG:4326:1', 'EPSG:4326:2', 'EPSG:4326:3', 'EPSG:4326:4', 'EPSG:4326:5', 'EPSG:4326:6', 'EPSG:4326:7', 'EPSG:4326:8', 'EPSG:4326:9', 'EPSG:4326:10', 'EPSG:4326:11', 'EPSG:4326:12', 'EPSG:4326:13', 'EPSG:4326:14', 'EPSG:4326:15', 'EPSG:4326:16', 'EPSG:4326:17', 'EPSG:4326:18', 'EPSG:4326:19', 'EPSG:4326:20', 'EPSG:4326:21'],
      format: 'application/vnd.mapbox-vector-tile',
      layerName: 'ne:pl_roads',
      resolutions: [0.703125, 0.3515625, 0.17578125, 0.087890625, 0.0439453125, 0.02197265625, 0.010986328125, 0.0054931640625, 0.00274658203125, 0.001373291015625, 6.866455078125E-4, 3.4332275390625E-4, 1.71661376953125E-4, 8.58306884765625E-5, 4.291534423828125E-5, 2.1457672119140625E-5, 1.0728836059570312E-5, 5.364418029785156E-6, 2.682209014892578E-6, 1.341104507446289E-6, 6.705522537231445E-7, 3.3527612686157227E-7],
      style: "",
      projection: new Projection({
        code: "EPSG:4326",
        units: "degrees",
        axisOrientation: "neu"
      }),
      lines: []

    }
  },
  methods: {
    constructSource() {
      var params = {
        'REQUEST': 'GetTile',
        'SERVICE': 'WMTS',
        'VERSION': '1.0.0',
        'LAYER': this.layerName,
        'STYLE': this.style,
        'TILEMATRIX': this.gridsetName + ':{z}',
        'TILEMATRIXSET': this.gridsetName,
        'FORMAT': this.format,
        'TILECOL': '{x}',
        'TILEROW': '{y}'
      };
      var url = this.baseUrl + '?'
      for (var param in params) {
        url = url + param + '=' + params[param] + '&';
      }
      url = url.slice(0, -1);

      let _that = this;

      var source = new VectorTileSource({
        url: url,
        format: new MVT({}),
        projection: this.projection,
        tileGrid: new WMTS({
          tileSize: [256, 256],
          origin: [-180.0, 90.0],
          resolutions: _that.resolutions,
          matrixIds: _that.gridNames
        }),
        wrapX: true
      });
      return source;
    },
    doLineSegmentsIntersect(line1, line2) {
      const coords1 = line1.getGeometry().getOrientedFlatCoordinates();
      const coords2 = line2.getGeometry().getOrientedFlatCoordinates();

      const p1 = coords1[0];
      const p2 = coords1[1];
      const p3 = coords2[0];
      const p4 = coords2[1];

      // 计算方向
      function direction(p, q, r) {
        return (r[1] - p[1]) * (q[0] - p[0]) - (q[1] - p[1]) * (r[0] - p[0]);
      }

      // 检查点 r 是否在线段 pq 上
      function onSegment(p, q, r) {
        return Math.min(p[0], q[0]) <= r[0] && r[0] <= Math.max(p[0], q[0]) &&
          Math.min(p[1], q[1]) <= r[1] && r[1] <= Math.max(p[1], q[1]);
      }

      const d1 = direction(p3, p4, p1);
      const d2 = direction(p3, p4, p2);
      const d3 = direction(p1, p2, p3);
      const d4 = direction(p1, p2, p4);

      // 线段相交的条件：
      if (((d1 > 0 && d2 < 0) || (d1 < 0 && d2 > 0)) &&
        ((d3 > 0 && d4 < 0) || (d3 < 0 && d4 > 0))) {
        return true; // 线段相交
      }

      // 检查是否有点在另一条线段上
      if (d1 === 0 && onSegment(p3, p4, p1)) return true;
      if (d2 === 0 && onSegment(p3, p4, p2)) return true;
      if (d3 === 0 && onSegment(p1, p2, p3)) return true;
      if (d4 === 0 && onSegment(p1, p2, p4)) return true;

      return false; // 线段不相交
    }
  },
  mounted() {
    console.log('server start...');

    const style = new Style({
      fill: new Fill({
        color: 'rgba(255, 255, 255, 0.6)',
      }),
      stroke: new Stroke({
        color: '#319FD3',
        width: 1,
      })
    });

    let _that = this;

    const layer = new VectorTile({
      source: this.constructSource(),
      declutter: true,
      style: function (feature) {
        let name = feature.get("name");
        if (name === "科华南路") {
          style.getStroke().setWidth(3);
          style.getStroke().setColor("#c699ec");
          let osm_id = feature.get("osm_id")
          if (osm_id === 170435804) {
            _that.lines.push(feature.getGeometry())
            if (_that.lines.length == 2) {
              let intersect = _that.doLineSegmentsIntersect(_that.lines[0], _that.lines[1])
              console.log(`是否相交: ${intersect}`)
              _that.lines = [];
            }
          }
          console.log(feature)
        } else {
          style.getStroke().setWidth(1);
          style.getStroke().setColor("#319FD3");
        }
        return style;
      }
    })
    const view = new View({
      center: [0, 0],
      zoom: 2,
      resolutions: this.resolutions,
      projection: this.projection,
      extent: [-180.0, -90.0, 180.0, 90.0]
    });

    const debugLayer = new TileLayer({
      source: new TileDebug({
        template: 'z:{z} x:{x} y:{y}',
        projection: layer.getSource().getProjection(),
        tileGrid: layer.getSource().getTileGrid(),
        zDirection: 1,
      }),
    });

    map.value = new Map({
      layers: [layer, debugLayer],
      target: "map",
      view,
      controls: defaultControls({ attribution: false }).extend([

      ])
    })

    map.value.getView().fit([103.9300086, 30.4990015, 104.2289856, 30.6759994], map.value.getSize());
  }
}

</script>

<style scoped>
#map {
  width: 1920px;
  height: 1080px;
}
</style>
