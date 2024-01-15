<template>
    <div :class="boxView ? 'box-item active' : 'box-item'">
        <span class="move-icon"><slot></slot></span>
        <el-switch v-model="layerStatus" :active-color="color"></el-switch>

        <span>
            <b>{{ nameLayer != '' ? nameLayer.length > 18 ? nameLayer.slice(0, 18) + ' ...' : nameLayer : title }}</b>
            <span v-show="layerStatus">
                <button class="btn-view" @click="boxView =! boxView">
                    <md-icon>settings</md-icon>
                </button>
            </span>
        </span>


        <!-- Propriedades e Funções de uma Camada - Invocadas quando se clica na engrenagem de uma camada ativa 
        Todos o ícones são importados do Material Desing do Google Usando o comando <md-icon>nome_icon</md-icon> 
        Para alterar os nome bastas ir nos arquivos english.js e prtuguese.js-->

        <div v-show="boxView && layerStatus" class="box-options">

            <!-- Desabilitar - Desabilita a camadada sem precisar abrir a tela de Adicionar camadas -->

            <el-tooltip effect="dark"
                    :content="$t('map.sidebarLayer.options.delete')"
                    placement="top-end">
                <md-icon @click.native="disabled()">delete</md-icon>
            </el-tooltip>

            <!-- Zoom - Centraliza a camada na tela, sejam regiões ou localidades -->

            <el-tooltip effect="dark"
                    :content="$t('map.sidebarLayer.options.zoom')"
                    placement="top-end">
                <md-icon @click.native="extend()">center_focus_strong</md-icon>
            </el-tooltip>

            <!-- Informação da Camada - descrição aprofundada sobre a camada. Abre uma nova tela (componente) -->

            <el-tooltip effect="dark"
                    :content="$t('map.sidebarLayer.options.infosLayer')"
                    placement="top-end">
                <md-icon @click.native="infosLayer()">assignment</md-icon>
            </el-tooltip>

            <!-- Informação sobre os objetos - Funcionalidade que, ao habilitada, permite a seleção de objetos 
                da camada e apresenta informações específicas do objeto selecionado -->

            <el-tooltip effect="dark"
                    :content="$t('map.sidebarLayer.options.infosVector')"
                    placement="top-end">
                <md-icon :class="getInfo ? 'active' : ''" @click.native="setStatusGetInfo()">info</md-icon>
            </el-tooltip>

            <!-- Mudar a cor - Alterar a cor dos objetos da camada -->

            <el-tooltip effect="dark"
                    :content="$t('map.sidebarLayer.options.editColor')"
                    placement="top-end">
                    <el-color-picker v-model="colorVector" show-alpha size="medium"></el-color-picker>
            </el-tooltip>

            <!-- Baixar .shp - baixar o arquivo da camada (formato .shp) -->

            <el-tooltip effect="dark"
                    :content="$t('map.sidebarLayer.options.download')"
                    placement="top-end">
                <md-icon @click.native="downloadSHP()">save_alt</md-icon>
            </el-tooltip>
        </div>

        <div id="popup" class="ol-popup"></div>
    </div>
</template>



<!-- ############ SCRIPT- Lógica Da Página ############ -->



<script>

// Importações necessárias do Add Layer para permitir a deleção
import axios from 'axios'

import {
  overlayGroup
} from '@/views/assets/js/map/overlayGroup'
//-------------------------------------------------

import { mapState } from 'vuex'
import {
    emptyStyle,
    lineStyle,
    polygonStyle,
    pointStyle
} from '@/views/assets/js/map/Styles'
import Map from '@/middleware/Map'
import Dashboard from '@/middleware/Dashboard'

export default {
    props: {
        status: Boolean,
        color: String,
        titleInit: {
            type: String,
            required: false
        },
        id: Number,
        group: Object
    },
    computed: {
        ...mapState('map', ['idInfoFeatureLayer'])
    },
    data() {
        return {
            boxView: false,
            title: '',
            type: '',
            layerStatus: true,
            colorVector: null,
            getInfo: false,
            select: null,
            overlay: null,
            nameLayer: '',

            //Data para fazer a desativação da camada
            listLayers: [],
            allLayers: [],
            btnDisabled: true,

        }
    },
    async mounted() {
        this.layerStatus = this.status
        this.title = this.titleInit

        if(this.id != undefined) {
            let layers = await Map.getLayers('layer_id=' + this.id)
            this.nameLayer = layers.data.features[0].properties.name
            this.title = layers.data.features[0].properties.f_table_name
        }

        this.getColor()


        //MONTANDO os arrays allLayers e listLayers
        try {
            let result = await Map.getLayers(null)
            this.allLayers = result.data.features

            // initialize the list of layers with all available layers
            this.listLayers = this.allLayers

        }catch (error) {
            this.$alert(this.$t('map.addLayer.msg.errMsg'), this.$t('map.addLayer.msg.errTitle'), {
                confirmButtonText: 'OK',
                type: 'error'
            })
        }
    },
    watch: {
        // active and disable the layer
        layerStatus(val) {
          this.group.getLayers().forEach(sublayer => {
            if (sublayer.get('title') === this.title)
              sublayer.setVisible(this.layerStatus)
          })
        },
        //edit color of layer
        colorVector(val) {
            if(val == null)
                this.colorVector = "rgba(255,255,255,0)"
            else
                this.group.getLayers().forEach(sublayer => {
                    if (sublayer.get('title') === this.title && this.layerStatus) {

                        let newStyle = null
                        if(this.type == 'multilinestring' || this.type == 'linestring') {
                            newStyle = new ol.style.Style({
                                stroke: new ol.style.Stroke({
                                    color: val,
                                    width: 3
                                })
                            })
                        } else if(this.type == 'multipoint' || this.type == 'point') {
                            newStyle = new ol.style.Style({
                                image: new ol.style.Circle({
                                    radius: 8,
                                    stroke: new ol.style.Stroke({
                                        color: 'white',
                                        width: 2
                                    }),
                                    fill: new ol.style.Fill({
                                        color: val
                                    })
                                })
                            })
                        } else if(this.type == 'multipolygon' || this.type == 'polygon') {
                            newStyle = new ol.style.Style({
                                stroke: new ol.style.Stroke({
                                    color: '#000000',
                                    width: 3
                                }),
                                fill: new ol.style.Fill({
                                    color: val
                                })
                            })
                        }

                        sublayer.getSource().getFeatures().map(feature => {
                          if(feature.getStyle() !== emptyStyle)
                            feature.setStyle(newStyle)
                        });
                        sublayer.setStyle(newStyle)
                    }
                })
        },
        //active 'info attrib' feature
        idInfoFeatureLayer(val){
          if(val == this.title) {
            this.getInfo = true
            this._getInfosFeatures()
          } else {
            this.getInfo = false
            this._clearInteractions()
          }
        }
    },
    methods: {
        async getColor() {
          let response = await Dashboard.getFeatureTable(this.title)

          for (let layer of this.group.getLayers().getArray()) {

            if (layer.get('title') === this.title) {
              this.type = response.data.features[0].geometry.type.toLowerCase()
              let styleType = typeof(layer.getStyle())

              if (this.type === 'multilinestring' || this.type === 'linestring') {
                if (styleType === 'function')
                  layer.setStyle(lineStyle)

                this.colorVector = lineStyle.getStroke().getColor()

              } else if (this.type === 'multipoint' || this.type === 'point') {
                if (styleType === 'function')
                  layer.setStyle(pointStyle)

                this.colorVector = pointStyle.getImage().getFill().getColor()

              } else if (this.type === 'multipolygon' || this.type === 'polygon') {
                if (styleType === 'function')
                  layer.setStyle(polygonStyle)

                this.colorVector = polygonStyle.getFill().getColor()
              }

              break;
            }

          }
        },

        //Deastivar Camada
		disabled() {
            //if(this.btnDisabled == false)
            //    this.btnDisabled = true

            overlayGroup.getLayers().forEach(sublayer => {
                if(sublayer != undefined && sublayer.get('id') != undefined && sublayer.get('id') == this.id) {
                    overlayGroup.getLayers().remove(sublayer)
                    this.$store.dispatch('map/setRemoveLayers', this.id)
                    this.btnDisabled = false
                    return true
                }
            })
        },
		
		
		
		// Centraliza a tela para a camada selecionada

        extend(){
            this.group.getLayers().forEach(sublayer => {
                if (sublayer.get('title') === this.title && this.layerStatus) {
                    let extentLayer = sublayer.getSource().getExtent();
                    let extentEmpty = ol.extent.createEmpty();

                    ol.extent.extend(extentEmpty, extentLayer);
                    this.$root.olmap.getView().fit(extentEmpty, this.$root.olmap.getSize());
                }
            })
        },
        infosLayer(){
            if(this.id != null && this.id !== undefined) {
                this.$store.dispatch('map/setBoxInfoVector', false)
                this.$store.dispatch('map/setBoxGeocoding', false)
                this.$store.dispatch('map/setBoxNotifications', false)

                this.$store.dispatch('map/setBoxInfoLayer', true)
                this.$store.dispatch('map/setIdInfoLayer', this.id)
            } else {
                this.$store.dispatch('map/setBoxInfoLayer', false)
                this.$store.dispatch('map/setIdInfoLayer', null)
            }
        },
        setStatusGetInfo() {
            if(this.getInfo == true)
              this.$store.dispatch('map/setIdInfoFeatureLayer', null)
            else
              this.$store.dispatch('map/setIdInfoFeatureLayer', this.title)
        },
        downloadSHP() {
            let link = process.env.urlGeoserverPauliceia + '/ows?service=WFS&version=1.0.0&request=GetFeature&typeName=pauliceia:'+this.title.toLowerCase()+'&outputFormat=SHAPE-ZIP'
            window.open(link, '_blank')
        },
        _getInfosFeatures(){
          let vm = this

          // clean the map interactions
          vm.$root.olmap.getOverlays().clear()

          // create popup
          $("#popup").append(
            `<div id="popup-${vm.title}" title="information of vector">
              <div id="popup-content-${vm.title}"></div>
            </div>`
          )

          // select popup
          vm.containerPopup = document.getElementById(`popup-${vm.title}`)
          vm.contentPopup = document.getElementById(`popup-content-${vm.title}`)

          $(vm.containerPopup).css({
              "display": "block",
              "position": "absolute",
              "background-color": "white",
              "-webkit-filter": "drop-shadow(0 1px 4px rgba(0,0,0,0.2))",
              "filter": "drop-shadow(0 1px 4px rgba(0,0,0,0.2))",
              "padding": "15px",
              "border-radius": "10px",
              "border": "1px solid #cccccc",
              "bottom": "12px",
              "left": "-50px",
              "min-width": "280px"
          })
          $(vm.containerPopup).addClass("ol-popup")

          // create interaction and overlay
          vm.select = new ol.interaction.Select()
          vm.overlay = new ol.Overlay({
              element: vm.containerPopup,
              autoPan: true
          })
          vm.$root.olmap.addInteraction(vm.select)
          vm.$root.olmap.addOverlay(vm.overlay)

          // when the user clicks on the feature
          vm.select.on('select', event => {
            vm.overlay.setPosition(undefined)

            event.selected.filter(
              feature => ((feature.getId().split('.'))[0]) == vm.title.toLowerCase()
            ).forEach(feat => {
              feat.setStyle()
              let coordinate = feat.getGeometry().getFirstCoordinate();

              vm.contentPopup.innerHTML = ''
              vm.contentPopup.innerHTML = `<p style="margin:0; padding:0;"><strong>id:</strong> ${feat.getId().split('.')[1]} </p>`

              for (const [key, value] of Object.entries(feat.getProperties())) {
                // do not add to the info box the `changeset_id` and `version` properties
                if (key === 'changeset_id' || key === 'version')
                  continue

                if (typeof(value) !== 'object')
                  vm.contentPopup.innerHTML += `<p style="margin:0; padding:0;"><strong>${key}:</strong> ${value} </p>`
              }

              vm.overlay.setPosition(coordinate)
            })
          });
        },
        _clearInteractions(){
            this.$root.olmap.removeInteraction(this.select)
            this.$root.olmap.removeOverlay(this.overlay)
            this.select = null
            this.overlay = null
        }
    }
}
</script>

<style lang="sass" scoped>
.box-item
    margin-top: 5px
    padding: 5px

    .move-icon
        cursor: pointer
        margin: 0
        padding: 3px
    .move-icon:hover
        color: #f15a29

    span
        padding-left: 7.5px
        font-size: 1.2em

        .btn-view
            background: none
            border: none
            color: #FFF
            cursor: pointer
            float: right
            margin-top: -1px

        .btn-view:hover
            color: #CCC
            text-shadow: 0px 0px 3px #333

    .box-options
        width: 100%
        margin-top: 5px
        background: rgba(#FFF, 0.3)
        padding: 8px 3px
        display: flex
        align-items: center

        .md-icon
            cursor: pointer
        .md-icon:hover, .md-icon.active
            color: #f15a29

        .btn-color
            margin: 0 10px

.box-item.active
    background: rgba(#FFF, 0.2)

//POPUP
.ol-popup
    position: absolute
    background-color: white
    display: none
    -webkit-filter: drop-shadow(0 1px 4px rgba(0,0,0,0.2))
    filter: drop-shadow(0 1px 4px rgba(0,0,0,0.2))
    padding: 15px
    border-radius: 10px
    border: 1px solid #cccccc
    bottom: 12px
    left: -50px
    min-width: 280px

.ol-popup:after, .ol-popup:before
    top: 100%
    border: solid transparent
    content: " "
    height: 0
    width: 0
    position: absolute
    pointer-events: none

.ol-popup:after
    border-top-color: white
    border-width: 10px
    left: 48px
    margin-left: -10px

.ol-popup:before
    border-top-color: #cccccc
    border-width: 11px
    left: 48px
    margin-left: -11px

</style>
