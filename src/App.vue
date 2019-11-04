<template>
  <div id="app" :class="[$options.name]">
    <!-- app map -->
    <vl-map v-if="mapVisible" class="map" ref="map" :load-tiles-while-animating="true" :load-tiles-while-interacting="true"
            @click="clickCoordinate = $event.coordinate" @postcompose="onMapPostCompose"
            data-projection="EPSG:4326" @mounted="onMapMounted">
      <!-- map view aka ol.View -->
      <vl-view ref="view" :center.sync="center" :zoom.sync="zoom" :rotation.sync="rotation"></vl-view>

      <!-- interactions -->
      <vl-interaction-select :features.sync="selectedFeatures">
        <template slot-scope="select">
          <!-- select styles -->
          <vl-style-box>
            <vl-style-stroke color="#423e9e" :width="7"></vl-style-stroke>
            <vl-style-fill :color="[254, 178, 76, 0.7]"></vl-style-fill>
            <vl-style-circle :radius="5">
              <vl-style-stroke color="#423e9e" :width="7"></vl-style-stroke>
              <vl-style-fill :color="[254, 178, 76, 0.7]"></vl-style-fill>
            </vl-style-circle>
          </vl-style-box>
          <vl-style-box :z-index="1">
            <vl-style-stroke color="#d43f45" :width="2"></vl-style-stroke>
            <vl-style-circle :radius="5">
              <vl-style-stroke color="#d43f45" :width="2"></vl-style-stroke>
            </vl-style-circle>
          </vl-style-box>
          <!--// select styles -->

          <!-- selected feature popup -->
          <!--
          <vl-overlay class="feature-popup" v-for="feature in select.features" :key="feature.id" :id="feature.id"
                      :position="pointOnSurface(feature.geometry)" :auto-pan="true" :auto-pan-animation="{ duration: 300 }">
            <template slot-scope="popup">
              <section class="card">
                <header class="card-header">
                  <p class="card-header-title">
                    Feature ID {{ feature.id }}
                  </p>
                  <a class="card-header-icon" title="Close"
                     @click="selectedFeatures = selectedFeatures.filter(f => f.id !== feature.id)">
                    <b-icon icon="close"></b-icon>
                  </a>
                </header>
                <div class="card-content">
                  <div class="content">
                    <p>
                      Overlay popup content for Feature with ID <strong>{{ feature.id }}</strong>
                    </p>
                    <p>
                      Popup: {{ JSON.stringify(popup) }}
                    </p>
                    <p>
                      Feature: {{ JSON.stringify({ id: feature.id, properties: feature.properties }) }}
                    </p>
                  </div>
                </div>
              </section>
            </template>
          </vl-overlay>
          -->
          <!--// selected popup -->
        </template>
      </vl-interaction-select>
      <vl-interaction-translate
          :features="selectedFeatures"
          @translatestart="beginCoord"
          @translateend="endCoord"
      ></vl-interaction-translate>
      <!--// interactions -->

      <!-- other layers from config -->
      <component v-for="layer in layers" :is="layer.cmp" v-if="layer.visible" :key="layer.id" v-bind="layer">
        <!-- add vl-source-* -->
        <component :is="layer.source.cmp" v-bind="layer.source">
          <!-- add static features to vl-source-vector if provided -->
          <vl-feature v-if="layer.source.staticFeatures && layer.source.staticFeatures.length"
                      v-for="feature in layer.source.staticFeatures" :key="feature.id"
                      :id="feature.id" :properties="feature.properties">
            <component :is="geometryTypeToCmpName(feature.geometry.type)" v-bind="feature.geometry"></component>
          </vl-feature>

          <!-- add inner source if provided (like vl-source-vector inside vl-source-cluster) -->
          <component v-if="layer.source.source" :is="layer.source.source.cmp" v-bind="layer.source.source">
            <!-- add static features to vl-source-vector if provided -->
            <vl-feature v-if="layer.source.source.staticFeatures && layer.source.source.staticFeatures.length"
                        v-for="feature in layer.source.source.staticFeatures" :key="feature.id"
                        :id="feature.id" :properties="feature.properties">
              <component :is="geometryTypeToCmpName(feature.geometry.type)" v-bind="feature.geometry"></component>
            </vl-feature>
          </component>
        </component>
        <!--// vl-source-* -->

        <!-- add style components if provided -->
        <!-- create vl-style-box or vl-style-func -->
        <component v-if="layer.style" v-for="(style, i) in layer.style" :key="i" :is="style.cmp" v-bind="style">
          <!-- create inner style components: vl-style-circle, vl-style-icon, vl-style-fill, vl-style-stroke & etc -->
          <component v-if="style.styles" v-for="(st, cmp) in style.styles" :key="cmp" :is="cmp" v-bind="st">
            <!-- vl-style-fill, vl-style-stroke if provided -->
            <vl-style-fill v-if="st.fill" v-bind="st.fill"></vl-style-fill>
            <vl-style-stroke v-if="st.stroke" v-bind="st.stroke"></vl-style-stroke>
          </component>
        </component>
        <!--// style -->
      </component>
      <!--// other layers -->
    </vl-map>
    <!--// app map -->

    <!-- map panel, controls -->
    <div class="map-panel">
      <b-collapse class="panel box is-paddingless" :open.sync="panelOpen">
        <div slot="trigger" class="panel-heading">
          <div class="columns">
            <div class="column is-11">
              <strong>Map panel</strong>
            </div>
            <div class="column">
              <b-icon :icon="panelOpen ? 'chevron-up' : 'chevron-down'" size="is-small"></b-icon>
            </div>
          </div>
        </div>
        <p class="panel-tabs">
          <a @click="showMapPanelTab('state')" :class="mapPanelTabClasses('state')">State</a>
          <a @click="showMapPanelTab('layers')" :class="mapPanelTabClasses('layers')">Layers</a>
        </p>

        <div class="panel-block" v-show="mapPanel.tab === 'state'">
          <table class="table is-fullwidth">
            <tr>
              <th>Map center</th>
              <td>{{ center }}</td>
            </tr>
            <tr>
              <th>Map zoom</th>
              <td>{{ zoom }}</td>
            </tr>
            <tr>
              <th>Selected features</th>
              <td>{{ selectedFeatures.map(f => f.id) }}</td>
            </tr>
          </table>
        </div>

        <div class="panel-block" v-for="layer in layers" :key="layer.id" @click="showMapPanelLayer"
             :class="{ 'is-active': layer.visible }"
             v-show="mapPanel.tab === 'layers'">
             <b-switch :key="layer.id" v-model="layer.visible">
            {{ layer.title }}
          </b-switch>
        </div>
      </b-collapse>
    </div>
    <!--// map panel, controls -->
  </div>
</template>

<script>
  import { kebabCase, camelCase } from 'lodash'
  import { addProj, getProj, findPointOnSurface, createStyle } from 'vuelayers/lib/ol-ext'
  import ScaleLine from 'ol/control/ScaleLine'
  import FullScreen from 'ol/control/FullScreen'
  import ZoomSlider from 'ol/control/ZoomSlider'
  import { register } from 'ol/proj/proj4.js'
  import proj4 from 'proj4/dist/proj4-src.js'
  import pointsJson from './assets/point.geojson'
  import polygonsJson from './assets/polygon.geojson'

  // EPSG:5514, transformace EPSG:1623 - ČR, přesnost 1m (odpovídá definici EPSG:102067, kterou používáš) towgs84 udává transformaci z S-JTSK do ETRS89 (European Terrestrial Reference System 1989)
  proj4.defs('EPSG:5514,EPSG:1623', '+proj=krovak +lat_0=49.5 +lon_0=24.83333333333333 +alpha=30.28813972222222 +k=0.9999 +x_0=0 +y_0=0 +ellps=bessel +towgs84=570.8,85.7,462.8,4.998,1.587,5.261,3.56 +units=m +no_defs')
  // EPSG:5514, transformace EPSG:5239 - ČR, přesnost 1m towgs84 udává transformaci z S-JTSK/05 do WGS 84
  proj4.defs('EPSG:5514,EPSG:5239', '+proj=krovak +lat_0=49.5 +lon_0=24.83333333333333 +alpha=30.28813972222222 +k=0.9999 +x_0=0 +y_0=0 +ellps=bessel +towgs84=572.213,85.334,461.94,-4.9732,-1.529,-5.2484,3.5378 +units=m +no_defs')
  // EPSG:5514, transformace EPSG:15965 - ČR a SR, přesnost 6m
  proj4.defs('EPSG:5514,EPSG:15965', '+proj=krovak +lat_0=49.5 +lon_0=24.83333333333333 +alpha=30.28813972222222 +k=0.9999 +x_0=0 +y_0=0 +ellps=bessel +towgs84=589,76,480,0,0,0,0 +units=m +no_defs')
  // EPSG:5514, transformace EPSG:4836 - SR, přesnost 1m (dodávám pro úplnost) towgs84 udává transformaci z S-JTSK do ETRS89
  proj4.defs('EPSG:5514,EPSG:4836', '+proj=krovak +lat_0=49.5 +lon_0=24.83333333333333 +alpha=30.28813972222222 +k=0.9999 +x_0=0 +y_0=0 +ellps=bessel +towgs84=485,169.5,483.8,7.786,4.398,4.103,0 +units=m +no_defs')

  register(proj4)
  addProj(getProj('EPSG:5514,EPSG:1623'))
  addProj(getProj('EPSG:5514,EPSG:5239'))
  addProj(getProj('EPSG:5514,EPSG:15965'))
  addProj(getProj('EPSG:5514,EPSG:4836'))

  const easeInOut = t => 1 - Math.pow(1 - t, 3)

  const methods = {
    camelCase,
    pointOnSurface: findPointOnSurface,
    geometryTypeToCmpName (type) {
      return 'vl-geom-' + kebabCase(type)
    },
    selectFilter (feature) {
      return ['position-feature'].indexOf(feature.getId()) === -1
    },
    onMapPostCompose ({ vectorContext, frameState }) {
      if (!this.$refs.marker || !this.$refs.marker.$feature) return

      const feature = this.$refs.marker.$feature
      if (!feature.getGeometry() || !feature.getStyle()) return

      const flashGeom = feature.getGeometry().clone()
      const duration = feature.get('duration')
      const elapsed = frameState.time - feature.get('start')
      const elapsedRatio = elapsed / duration
      const radius = easeInOut(elapsedRatio) * 35 + 5
      const opacity = easeInOut(1 - elapsedRatio)
      const fillOpacity = easeInOut(0.5 - elapsedRatio)

      vectorContext.setStyle(createStyle({
        imageRadius: radius,
        fillColor: [119, 170, 203, fillOpacity],
        strokeColor: [119, 170, 203, opacity],
        strokeWidth: 2 + opacity,
      }))

      vectorContext.drawGeometry(flashGeom)
      vectorContext.setStyle(feature.getStyle()(feature)[0])
      vectorContext.drawGeometry(feature.getGeometry())

      if (elapsed > duration) {
        feature.set('start', Date.now())
      }

      this.$refs.map.render()
    },
    onMapMounted () {
      // now ol.Map instance is ready and we can work with it directly
      this.$refs.map.$map.getControls().extend([
        new ScaleLine(),
        new FullScreen(),
        new ZoomSlider(),
      ])
    },
    // base layers
    // map panel
    mapPanelTabClasses (tab) {
      return {
        'is-active': this.mapPanel.tab === tab,
      }
    },
    showMapPanelLayer (layer) {
      layer.visible = !layer.visible
    },
    showMapPanelTab (tab) {
      this.mapPanel.tab = tab
      if (tab !== 'draw') {
        this.drawType = undefined
      }
    },
    beginCoord (event) {
      console.log(event.coordinate)
    },
    endCoord (event) {
      console.log(event.coordinate)
    },
  }

  export default {
    name: 'vld-demo-app',
    methods,
    data () {
      return {
        center: [ 12.73980021710448, 50.205437881007214 ],
        // center: [ 14.828796386718748, 49.71027258210569 ],
        zoom: 15,
        // zoom: 9,
        rotation: 0,
        clickCoordinate: undefined,
        selectedFeatures: [],
        deviceCoordinate: undefined,
        mapPanel: {
          tab: 'layers',
        },
        panelOpen: true,
        mapVisible: true,
        // layers config
        layers: [
          {
            id: 'osm',
            cmp: 'vl-layer-tile',
            title: 'OpenStreetMap',
            visible: false,
            source: {
              cmp: 'vl-source-osm',
            },
          },
          // mapy.cz
          {
            id: 'wmts-mapy-cz',
            title: 'mapy.cz',
            cmp: 'vl-layer-tile',
            visible: true,
            source: {
              cmp: 'vl-source-wmts',
              url: 'https://mapserver.mapy.cz/base-m/{TileMatrix}-{TileCol}-{TileRow}',
              attribution: '© Seznam.cz, a.s., © AOPK ČR – ochrana přírody a krajiny, © Slovenská agentúra živ. prostredia“ - digitální model terénu SR, © Národné lesnické centrum SR“ - lesní a polní cesty SR, © OpenStreetMap Contributors (jen mimo ČR a SR)', // nefunguje - potřeba zjistit proč
              layerName: 'base-m',
              matrixSet: 'EPSG:3857',
              format: 'image/jpeg',
              styleName: 'default',
              requestEncoding: 'REST',
            },
          },
          // Features for translation (points)
          {
            id: 'points',
            title: 'Body',
            cmp: 'vl-layer-vector',
            visible: false,
            renderMode: 'image',
            'z-index': 100,
            source: {
              cmp: 'vl-source-vector',
              projection: 'EPSG:5514,EPSG:15965',
              features: pointsJson.features,
            },
            style: [{
              cmp: 'vl-style-box',
              styles: {
                'vl-style-circle': {
                  radius: 10,
                  fill: {
                    color: 'rgba(255,0,0,0.8)',
                  },
                  stroke: {
                    color: '#FFFFFF',
                    width: 3,
                  },
                },
                'vl-style-stroke': {
                  width: 3,
                  color: '#000000',
                },
              },
            }],
          },
          // Features for translation (polygons)
          {
            id: 'polygons',
            title: 'Polygony',
            cmp: 'vl-layer-vector',
            visible: false,
            renderMode: 'image',
            'z-index': 99,
            source: {
              cmp: 'vl-source-vector',
              projection: 'EPSG:5514,EPSG:15965',
              features: polygonsJson.features,
            },
            style: [{
              cmp: 'vl-style-box',
              styles: {
                'vl-style-fill': {
                  fill: {
                    color: 'rgba(255,0,0,0.8)',
                  },
                  stroke: {
                    color: '#FFFFFF',
                    width: 3,
                  },
                },
              },
            }],
          },
        ],
      }
    },
  }
</script>

<style lang="sass">
  @import ~bulma/sass/utilities/_all

  html, body, #app
    width: 100%
    height: 100%
    margin: 0
    padding: 0

  .vld-demo-app
    position: relative

    .map
      height: 100%
      width: 100%

    .map-panel
      padding: 0

      .panel-heading
        box-shadow: 0 .25em .5em transparentize($dark, 0.8)

      .panel-content
        background: $white
        box-shadow: 0 .25em .5em transparentize($dark, 0.8)

      .panel-block
        &.draw-panel
          .buttons
          .button
            display: block
            flex: 1 1 100%

      +tablet()
        position: absolute
        top: 0
        right: 0
        max-height: 500px
        width: 22em

    .base-layers-panel
      position: absolute
      left: 50%
      bottom: 20px
      transform: translateX(-50%)

    .feature-popup
      position: absolute
      left: -50px
      bottom: 12px
      width: 20em
      max-width: none

      &:after, &:before
        top: 100%
        border: solid transparent
        content: ' '
        height: 0
        width: 0
        position: absolute
        pointer-events: none
      &:after
        border-top-color: white
        border-width: 10px
        left: 48px
        margin-left: -10px
      &:before
        border-top-color: #cccccc
        border-width: 11px
        left: 48px
        margin-left: -11px

      .card-content
        max-height: 20em
        overflow: auto

      .content
        word-break: break-all
</style>
