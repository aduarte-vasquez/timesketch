<!--
Copyright 2023 Google Inc. All rights reserved.
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->
<template>
  <v-card>
    <v-card-title>
      <v-img :src="getUnfurlLogo" max-height="32" contain class="mb-2"></v-img>
    </v-card-title>
    <v-card-subtitle class="pt-1">
      <span><b>Input:</b> {{ url }}</span>
    </v-card-subtitle>
    <v-card-text>
      <div v-show="!unfurlReady">
        <v-progress-linear color="primary" indeterminate> </v-progress-linear>
      </div>
      <v-divider></v-divider>
      <v-container fluid v-show="unfurlReady">
        <!-- Cytoscape container -->
        <div style="font-size: medium" class="py-1 px-1">
          <span style="font-weight: bold">Selected node context: </span>
          <br />
          <span style="font-style: italic" v-html="nodeContext"></span>
        </div>
        <div ref="graphContainer" :style="{ height: canvasHeight, width: '100%' }">
          <v-row no-gutters>
            <v-col>
              <div
                ref="cy"
                width="100%"
                class="pa-2"
                :style="{ 'min-height': canvasHeight, border: '1px', 'border-style': 'solid' }"
              ></div>
            </v-col>
            <v-col cols="auto">
              <div class="iconWrapper">
                <v-tooltip top open-delay="500">
                  <template v-slot:activator="{ on }">
                    <v-btn icon v-on="on" @click="resizeCanvas()">
                      <v-icon>mdi-fit-to-page-outline</v-icon>
                    </v-btn>
                  </template>
                  <span>resize graph</span>
                </v-tooltip>
                <v-tooltip top open-delay="500">
                  <template v-slot:activator="{ on }">
                    <v-btn icon v-on="on" @click="zoomGraph('plus')">
                      <v-icon>mdi-plus</v-icon>
                    </v-btn>
                  </template>
                  <span>zoom in</span>
                </v-tooltip>
                <v-tooltip top open-delay="500">
                  <template v-slot:activator="{ on }">
                    <v-btn icon v-on="on" @click="zoomGraph('minus')">
                      <v-icon>mdi-minus</v-icon>
                    </v-btn>
                  </template>
                  <span>zoom out</span>
                </v-tooltip>
              </div>
            </v-col>
          </v-row>
        </div>
      </v-container>
      <span>Powered by <a href="https://github.com/obsidianforensics/unfurl" target="_blank">dfir-unfurl</a></span>
    </v-card-text>
    <v-card-actions>
      <v-spacer></v-spacer>
      <v-btn color="primary" text @click="clearAndCancel"> close </v-btn>
    </v-card-actions>
  </v-card>
</template>

<script>
import ApiClient from '../../utils/RestApiClient'
import cytoscape from 'cytoscape'
import dagre from 'cytoscape-dagre'

cytoscape.use(dagre)

export default {
  props: ['url'],
  data() {
    return {
      unfurlReady: false,
      unfurlData: {},
      canvasHeight: '400px',
      nodeContext: '',
      nodeContextDefault: 'Select a node in the graph below to get more information.',
      config: {
        style: [
          {
            selector: 'node',
            style: {
              shape: 'roundrectangle',
              width: 'data(labelWidth)',
              height: 'data(labelHeight)',
              label: 'data(label)',
              'font-size': '10',
              'text-outline-width': '0px',
              padding: '7px',
              'background-color': '#e0e0e0',
              'text-outline-color': '#e0e0e0',
              'text-wrap': 'wrap',
              'text-max-width': '12em',
              'text-halign': 'center',
              'text-valign': 'center',
              color: '#000000',
            },
          },
          {
            selector: 'node:selected',
            style: {
              'border-color': ' #000',
              'border-width': '2px',
            },
          },
          {
            selector: 'edge',
            style: {
              width: 1,
              'curve-style': 'bezier',
              'control-point-step-size': 70,
              'target-arrow-shape': 'triangle',
              'font-size': 11,
              'text-rotation': 'autorotate',
              'text-outline-width': 3,
              'text-outline-color': '#FFFFFF',
              label: 'data(label)',
            },
          },
          {
            selector: 'edge.highlight',
            style: {
              lineColor: 'grey',
              width: 3,
            },
          },
        ],

        layout: {
          name: 'dagre',
          animated: false,
          prelayout: false,
          spacingFactor: 1,
        },
      },
    }
  },
  computed: {
    nodeContextValue() {
      return this.nodeContext
    },
    getUnfurlLogo() {
      if (this.$vuetify.theme.dark) {
        return '/unfurl-logo-dark.png'
      } else {
        return '/unfurl-logo.png'
      }
    },
  },
  methods: {
    clearAndCancel: function () {
      this.$emit('cancel')
    },
    getUnfurlData: function (url) {
      ApiClient.getUnfurlGraph(url)
        .then((response) => {
          this.unfurlData = response.data
          this.unfurlReady = true
          this.buildUnfurlGraph(this.unfurlData)
        })
        .catch((e) => {
          console.error('getting unfurl error: ', e)
        })
    },
    resizeCanvasWithDelay: function () {
      this.resizeTimeout = setTimeout(() => {
        this.resizeCanvas()
      }, 250)
    },
    resizeCanvas: function () {
      // Exit early if there is no graph canvas
      if (!this.$refs.graphContainer) {
        return
      }
      let canvasHeight = this.$refs.graphContainer.style.height
      let canvasWidth = this.$refs.graphContainer.style.width
      let canvas = this.$refs.cy
      canvas.style.minHeight = canvasHeight + 'px'
      canvas.style.height = canvasHeight + 'px'
      canvas.style.minWidth = canvasWidth + 'px'
      canvas.style.width = canvasWidth + 'px'
      this.cy.resize()
      this.cy.fit()
    },
    zoomGraph(direction) {
      const currentZoomLevel = this.cy.zoom()
      if (direction === 'plus') {
        this.cy.zoom({
          level: currentZoomLevel + 0.2,
        })
      }
      if (direction === 'minus') {
        this.cy.zoom({
          level: currentZoomLevel - 0.2,
        })
      }
    },
    buildUnfurlGraph: function (data = {}) {
      this.cy.elements().remove()

      let elements = []
      data.nodes.forEach((node) => {
        elements.push({
          group: 'nodes',
          data: {
            id: node.id,
            label: node.label,
            labelWidth: node.label.length * 6,
            labelHeight: 13,
            context: node.title,
          },
        })
      })
      data.edges.forEach((edge) => {
        elements.push({
          group: 'edges',
          data: {
            id: `e${edge.from}${edge.to}`,
            source: edge.from,
            target: edge.to,
            label: edge.label,
            context: edge.title,
          },
        })
      })

      this.cy.style(this.config.style)
      this.cy.add(elements)
      this.cy.layout(this.config.layout).run()
      this.resizeCanvasWithDelay()
    },
    nodeSelection: function (event) {
      this.cy.edges().removeClass('highlight')
      event.target.outgoers('edge').addClass('highlight')
      this.nodeContext = event.target.data().context
        ? event.target.data().context
        : 'No context available for this node.'
      this.resizeCanvas()
    },
  },
  mounted() {
    window.addEventListener('resize', this.resizeCanvasWithDelay)
    this.unfurlReady = false
    this.getUnfurlData(this.url)
    this.cy = cytoscape({
      container: this.$refs.cy,
      ...this.config,
    })
    this.cy.on('select', 'node', (event) => {
      this.nodeSelection(event)
    })
    this.cy.on('unselect', 'node', (event) => {
      this.cy.elements().removeClass('highlight')
      this.nodeContext = this.nodeContextDefault
    })
  },
}
</script>

<style scoped lang="scss">
.iconWrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: absolute;
}
</style>
