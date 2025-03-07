<template>
    <div id="map" 
        @mouseover="setMouseInside(true)" 
        @mouseleave="setMouseInside(false)">
        <Toolbar :tools="tools" ref="toolbar" />
        <div ref="map-container" class="map-container" :class="{'cursor-pointer': selectSingle,
                                         'cursor-crosshair': selectArea}"/>
    </div>
</template>

<script>
import 'ol/ol.css';
import {Map, View} from 'ol';
import {Tile as TileLayer, Vector as VectorLayer, Group as LayerGroup} from 'ol/layer';
import {OSM, Vector as VectorSource, Cluster} from 'ol/source';
import {defaults as defaultControls, Control} from 'ol/control';
import Collection from 'ol/Collection';
import {DragBox} from 'ol/interaction';
import {createEmpty as createEmptyExtent, isEmpty as isEmptyExtent, extend as extendExtent} from 'ol/extent';
import {transformExtent} from 'ol/proj';

import Feature from 'ol/Feature';
import { coordEach, coordAll } from '@turf/meta';
import bbox from '@turf/bbox';
import Point from 'ol/geom/Point';
import LineString from 'ol/geom/LineString';
import Polygon from 'ol/geom/Polygon';
import { fromExtent } from 'ol/geom/Polygon';

import ddb from 'ddb';
import HybridXYZ from '../classes/olHybridXYZ';
import Toolbar from './Toolbar.vue';
import Keyboard from '../keyboard';
import Mouse from '../mouse';
import { rootPath } from 'commonui/dynamic/pathutils';
import { clone } from 'commonui/classes/utils';

import {Circle as CircleStyle, Fill, Stroke, Style, Text, Icon} from 'ol/style';

export default {
  components: {
      Map, Toolbar
  },
  props: ['files'],
  data: function(){
      return {
        tools: [
            {
                id: 'select-features',
                title: "Select Features (CTRL)",
                icon: "hand pointer outline",
                exclusiveGroup: "select",
                onSelect: () => {
                    this.selectSingle = true;
                },
                onDeselect: () => {
                    this.selectSingle = false;
                }
            },
            {
                id: 'select-features-by-area',
                title: "Select Features by Area (CTRL+SHIFT)",
                icon: "square outline",
                exclusiveGroup: "select",
                onSelect: () => {
                    this.selectArea = true;
                    this.map.addInteraction(this.dragBox);
                },
                onDeselect: () => {
                    this.selectArea = false;
                    this.selectFeaturesByAreaKeyPressed = false;
                    this.map.removeInteraction(this.dragBox);
                }
            },
            {
                id: 'clear-selection',
                title: "Clear Selection (ESC)",
                icon: "ban",
                onClick: () => {
                    this.clearSelection();
                }
            },
            
        ],

        selectSingle: false,
        selectArea: false,
      };
  },
  mounted: function(){
    this._updateMap = null;
    this.lastFilesLoaded = null;

    const genShotStyle = (fill, stroke, size) => {
        let text = null;
        if (size > 1){
            text = new Text({
                text: size.toString(),
                font: 'bold 14px sans-serif',
                fill: new Fill({
                    color: 'rgba(252, 252, 255, 1)'
                })
            });
        }

        return new Style({
            image: new CircleStyle({
                radius: 8,
                fill: new Fill({
                    color: fill
                }),
                stroke: new Stroke({
                    color: stroke,
                    width: 2
                })
            }),
            text
        });
    };

    this.styles = {
        shot: feats => {
            let styleId = `_shot-${feats.length}`;
            let selected = false;

            for (let i = 0; i < feats.length; i++){
                if (feats[i].file && feats[i].file.selected){
                    styleId = `_shot-selected-${feats.length}`;
                    selected = true;
                    break;
                }
            }
            
            // Memoize
            if (!this.styles[styleId]){
                if (selected) this.styles[styleId] = genShotStyle('rgba(255, 158, 103, 1)', 'rgba(252, 252, 255, 1)', feats.length);
                else this.styles[styleId] = genShotStyle('rgba(75, 150, 243, 1)', 'rgba(252, 252, 255, 1)', feats.length);
            }

            return this.styles[styleId];
        },

        'shot-outlined': feats => {
            let styleId = `_shot-outlined-${feats.length}`;

            // Memoize
            if (!this.styles[styleId]){
                this.styles[styleId] = genShotStyle('rgba(253, 226, 147, 1)', 'rgba(252, 252, 255, 1)', feats.length);
            }

            return this.styles[styleId];
        },

        outline: new Style({
            stroke: new Stroke({
                color: 'rgba(253, 226, 147, 1)',
                width: 6
            })
        }),

        invisible: new Style({
            fill: new Fill({
                color: 'rgba(0, 0, 0, 0)'
            })
        }),

        startFlag: new Style({
            image: new Icon({
                anchor: [0.05, 1],
                anchorXUnits: 'fraction',
                anchorYUnits: 'fraction',
                src: rootPath('images/start-flag.svg'),
            })
        }),

        finishFlag: new Style({
            image: new Icon({
                anchor: [0.05, 1],
                anchorXUnits: 'fraction',
                anchorYUnits: 'fraction',
                src: rootPath('images/finish-flag.svg'),
            })
        }),

        flightPath: new Style({
            stroke: new Stroke({
                color: 'rgba(75, 150, 243, 1)', //'rgba(253, 226, 147, 1)',
                width: 4
            })
        })
    };

    this.fileFeatures = new VectorSource();
    const clusterSource = new Cluster({
        distance: 0.0001,
        source: this.fileFeatures
    });
    this.fileLayer = new VectorLayer({
        source: clusterSource,
        style: cluster => {
            const feats = cluster.get('features');
            const s = this.styles[feats[0].style];
            if (typeof s === "function") return s(feats);
            else return s;
        }
    });
    this.rasterLayer = new LayerGroup();

    this.extentsFeatures = new VectorSource();
    this.extentLayer = new VectorLayer({
        source: this.extentsFeatures,
        style: this.styles.invisible
    });

    this.outlineFeatures = new VectorSource();
    this.outlineLayer = new VectorLayer({
        source: this.outlineFeatures,
        style: this.styles.outline
    });
    this.footprintRastersLayer = new LayerGroup();

    this.flightPathFeatures = new VectorSource();
    this.flightPathLayer = new VectorLayer({
        source: this.flightPathFeatures
    });
    this.markerFeatures = new VectorSource();
    this.markerLayer = new VectorLayer({
        source: this.markerFeatures
    });

    this.topLayers = new LayerGroup();
    this.topLayers.setLayers(new Collection([
        this.flightPathLayer,
        this.fileLayer,
        this.markerLayer
    ]));

    this.dragBox = new DragBox({minArea: 0});
    this.dragBox.on('boxend', () => {
        let extent = this.dragBox.getGeometry().getExtent();

        // Select (default) or deselect (if all features are previously selected)
        const intersect = [];

        [this.fileFeatures, this.extentsFeatures].forEach(layer => {
            layer.forEachFeatureIntersectingExtent(extent, feat => {
                intersect.push(feat);
            });
        });

        let deselect = false;
        
        // Clear previous
        if (!Keyboard.isShiftPressed()){
            this.clearSelection();
        }else{
            deselect = intersect.length > 0 && intersect.filter(feat => feat.file.selected).length === intersect.length;
        }
        
        if (deselect){
            intersect.forEach(feat => feat.file.selected = false);
        }else{
            let scrolled = false;
            intersect.forEach(feat => {
                feat.file.selected = true;

                if (!scrolled){
                    this.$emit("scrollTo", feat.file);
                    scrolled = true;
                }
            });
        }
    });

    this.map = new Map({
        target: this.$refs['map-container'],
        layers: [
            new TileLayer({
                source: new OSM()
            }),
            this.rasterLayer,
            this.footprintRastersLayer,
            this.extentLayer,
            this.outlineLayer,
            this.topLayers,
        ],
        view: new View({
            center: [0, 0],
            zoom: 2
        })
    });

    const doSelectSingle = e => {
        let first = true;

        this.map.forEachFeatureAtPixel(e.pixel, feat => {
            // Only select the first entry
            if (!first) return;
            first = false;

            const feats = feat.get('features');
            if (feats){
                // Geoimage point cluster
                let selected = false;
                for (let i = 0; i < feats.length; i++){
                    if (feats[i].file && feats[i].file.selected){
                        selected = true;
                        break;
                    }
                }

                for (let i = 0; i < feats.length; i++){
                    if (feats[i].file) feats[i].file.selected = !selected;
                }
                
                // Inform other components we should scroll to this file
                if (!selected && feats.length && feats[0].file){
                    this.$emit("scrollTo", feats[0].file);
                }
            }else{
                // Extents selection
                if (feat.file) feat.file.selected = !feat.file.selected;
            }
        });
    };

    this.map.on('click', e => {
        // Single selection
        if (this.selectSingle){
            doSelectSingle(e);
        }else{
          // Remove all
          this.outlineFeatures.forEachFeature(outline => {
                this.outlineFeatures.removeFeature(outline);
                
                // Deselect style
                if (outline.feat.get('features')){
                    outline.feat.get('features').forEach(f => {
                        f.style = 'shot';
                    });
                }

                delete(outline.feat.outline);
          });
          this.clearLayerGroup(this.footprintRastersLayer);
          this.topLayers.setVisible(true);
        
          // Add selected
          this.map.forEachFeatureAtPixel(e.pixel, feat => {
            if (!feat.outline){
                let outline = null;

                if (feat.get('features')){
                    // Geoimage (point cluster)
                    const file = feat.get('features')[0].file;
                    if (file.entry.polygon_geom){
                        const coords = coordAll(file.entry.polygon_geom);
                        outline = new Feature(new Polygon([coords]));
                        outline.getGeometry().transform('EPSG:4326', 'EPSG:3857');
                    }
                    
                    // Nothing else to do
                    if (!outline) return;
                    
                    // Set style
                    feat.get('features').forEach(f => {
                        f.style = 'shot-outlined';
                    });

                    // Add geoprojected raster footprint
                    const rasterFootprint = new TileLayer({
                        extent: outline.getGeometry().getExtent(), 
                        source: new HybridXYZ({
                            url: file.path,
                            tileSize: 256,
                            transition: 0, // TODO: why transitions don't work?
                            minZoom: 14,
                            maxZoom: 22
                            // TODO: get min/max zoom somehow?
                        })
                    });
                    // tileLayer.file = f;
                    this.footprintRastersLayer.getLayers().push(rasterFootprint);
                }else{
                    // Extent
                    outline = feat;
                }

                this.outlineFeatures.addFeature(outline);

                // Add reference to self (for deletion later)
                outline.feat = feat;
                feat.outline = outline;

                this.topLayers.setVisible(false);
            }
          }, { 
              layerFilter: layer => {
                  return layer.getVisible() && 
                        (layer === this.fileLayer ||
                         layer === this.extentLayer);
              }
          });
          
          // Update styles
          this.fileLayer.changed();
        }
    });

    // Right click
    this.map.on('contextmenu', e => {
        // Single selection
        doSelectSingle(e);
    });

    Keyboard.onKeyDown(this.handleKeyDown);
    Keyboard.onKeyUp(this.handleKeyUp);

    // Redraw, otherwise openlayers does not draw
    // the map correctly
    setTimeout(() => this.map.updateSize(), 1);

    // Redraw when map is resized (via panels)
    this.$el.addEventListener('resized', () => {
        this.map.updateSize();
    });
  },
  beforeDestroy: function(){
    Keyboard.offKeyDown(this.handleKeyDown);
    Keyboard.offKeyUp(this.handleKeyUp);
  },
  watch: {
      files: {
        deep: true,
        handler: function(newVal, oldVal){
            // Prevent multiple updates
            if (this._updateMap){
                clearTimeout(this._updateMap);
                this._updateMap = null;
            }

            this.$log.info("Map.handler(newVal, oldVal)", clone(newVal), clone(oldVal));

            this._updateMap = setTimeout(() => {

                if (this.lastFilesLoaded == null) {
                    this.reloadFileLayers();
                    return;
                }

                // Do we need to redraw the features?
                // Count has changed or first or last items are different
                if (newVal.length !== this.lastFilesLoaded.length || 
                    newVal[0] !== this.lastFilesLoaded[0] || 
                    newVal[newVal.length - 1] !== this.lastFilesLoaded[this.lastFilesLoaded.length - 1]){
                   this.reloadFileLayers();
                }else{
                    // Just update (selection change)
                    this.fileLayer.changed();
                    this.updateRastersOpacity();
                }
                /*
                if (newVal.length !== oldVal.length || 
                    newVal[0] !== oldVal[0] || 
                    newVal[newVal.length - 1] !== oldVal[oldVal.length - 1]){
                   this.reloadFileLayers();
                }else{
                    // Just update (selection change)
                    this.fileLayer.changed();
                    this.updateRastersOpacity();
                }*/
            }, 5);
        }
      }
  },
  methods: {
      getSelectedFilesExtent: function(){
          const ext = createEmptyExtent();
          if (this.fileFeatures.getFeatures().length){
            extendExtent(ext, this.fileFeatures.getExtent());
          }
          this.rasterLayer.getLayers().forEach(layer => {
              extendExtent(ext, layer.getExtent());
          });
          return ext;
      },
      clearLayerGroup: function(layerGroup){
        let count = layerGroup.getLayers().getLength();
        for (var j = 0; j < count; j++) {
            let layer = layerGroup.getLayers().pop();
            layer = {};
        }
      },
      setMouseInside: function(flag){
          this.mouseInside = flag;
      },
      reloadFileLayers: function(){
        this.fileFeatures.clear();
        this.outlineFeatures.clear();
        this.flightPathFeatures.clear();
        this.markerFeatures.clear();
        this.clearLayerGroup(this.rasterLayer);

        const features = [];
        const rasters = this.rasterLayer.getLayers();

        let flightPath = [];

        // Create features, add them to map
        this.files.forEach(f => {
            if (!f.entry) return;

            if (f.entry.type === ddb.entry.type.GEOIMAGE || f.entry.type === ddb.entry.type.GEOVIDEO){
                const coords = coordAll(f.entry.point_geom)[0];
                const point = new Point(coords);
                const feat = new Feature(point);
                feat.style = 'shot';
                feat.file = f;
                feat.getGeometry().transform('EPSG:4326', 'EPSG:3857');
                features.push(feat);

                if (f.entry.type === ddb.entry.type.GEOIMAGE){
                    if (f.entry.meta.captureTime){
                        flightPath.push({
                            point,
                            captureTime: f.entry.meta.captureTime
                        });
                    }
                }
            }else if (f.entry.type === ddb.entry.type.GEORASTER){
                const extent = transformExtent(bbox(f.entry.polygon_geom), 'EPSG:4326', 'EPSG:3857');
                const tileLayer = new TileLayer({
                    extent, 
                    source: new HybridXYZ({
                        url: f.path,
                        tileSize: 256,
                        transition: 0, // TODO: why transitions don't work?
                        minZoom: 14,
                        maxZoom: 22
                        // TODO: get min/max zoom from file
                    })
                });
                tileLayer.file = f;
                rasters.push(tileLayer);

                // We create "hidden" extents as polygons
                // so that we can click raster layers
                const extentFeat = new Feature(fromExtent(extent));
                extentFeat.file = f;
                this.extentsFeatures.addFeature(extentFeat);
            }
        });

        this.lastFilesLoaded = clone(this.files);

        if (features.length) this.fileFeatures.addFeatures(features);

        // Add flight path line
        if (flightPath.length >= 2){
            flightPath.sort((a, b) => a.captureTime < b.captureTime ? -1 : (a.captureTime === b.captureTime ? 0 : 1));
            const startPoint = flightPath[0].point;
            const finishPoint = flightPath[flightPath.length - 1].point;

            const addFlag = (point, style) => {
                const feat = new Feature({
                    geometry: point
                });
                // feat.getGeometry().transform('EPSG:4326', 'EPSG:3857');
                feat.setStyle(style);
                this.markerFeatures.addFeature(feat);
            };

            addFlag(startPoint, this.styles.startFlag);
            addFlag(finishPoint, this.styles.finishFlag);

            // Draw linestring
            const pathFeat = new Feature({
                geometry: new LineString(flightPath.map(fp => fp.point.flatCoordinates))
            });
            pathFeat.setStyle(this.styles.flightPath);
            this.flightPathFeatures.addFeature(pathFeat);
        }

        const ext = this.getSelectedFilesExtent();
        if (!isEmptyExtent(ext)){
            // Zoom to it
            setTimeout(() => {
                this.map.getView().fit(ext, { 
                    padding: [40, 40, 40, 40], 
                    duration: 500,
                    minResolution: 0.5
                });
            }, 10);
        }

        this.updateRastersOpacity();
      },
      handleKeyDown: function(){
        if (Keyboard.isCtrlPressed() && this.mouseInside){
            if (Keyboard.isShiftPressed()){
                this.selectFeaturesByAreaKeyPressed = true;
                this.$refs.toolbar.selectTool('select-features-by-area');
            }else{
                this.$refs.toolbar.selectTool('select-features');
            }
        }
      },
      handleKeyUp: function(e){
        // ESC
        if (e.keyCode === 27){
            this.$refs.toolbar.selectTool('clear-selection');
            this.$refs.toolbar.deselectAll();
        }

        if (!Keyboard.isCtrlPressed() && this.mouseInside){
            if (!Keyboard.isShiftPressed() && this.selectFeaturesByAreaKeyPressed){
                this.$refs.toolbar.deselectTool('select-features-by-area');
            }
            this.$refs.toolbar.deselectTool('select-features');
        }
      },
      clearSelection: function(){
          this.files.forEach(f => f.selected = false);
      },
      updateRastersOpacity: function(){
          this.rasterLayer.getLayers().forEach(layer => {
              if (layer.file.selected) layer.setOpacity(0.8);
              else layer.setOpacity(1.0);
          });
      }
  }
}
</script>

<style scoped>
#map{
    border-top: 1px solid #030A03;
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
}
.map-container{
    width: 100%;
    height: 100%;

    &.cursor-pointer{
        cursor: pointer;
    }
    &.cursor-crosshair{
        cursor: crosshair;
    }
}
</style>