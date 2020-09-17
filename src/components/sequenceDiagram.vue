<template>
  <div id="myDiagramDiv" style="border: solid 1px black; width: 100%; height: 400px;"></div>
</template>

<script>
import go from 'gojs'
import {
  MessageLink,
  MessageDraggingTool,
  MessagingTool
} from './sequence'
const $ = go.GraphObject.make
let myDiagram = null

MessagingTool.prototype.insertLink = function (fromnode, fromport, tonode, toport) {
  var newlink = go.LinkingTool.prototype.insertLink.call(this, fromnode, fromport, tonode, toport)
  if (newlink !== null) {
    var model = this.diagram.model
    // specify the time of the message
    var start = this.temporaryLink.time
    var duration = 1
    newlink.data.time = start
    model.setDataProperty(newlink.data, 'text', 'msg')
    // and create a new Activity node data in the "to" group data
    var newact = {
      group: newlink.data.to,
      start: start,
      duration: duration
    }
    model.addNodeData(newact)
    // now make sure all Lifelines are long enough
    ensureLifelineHeights()
  }
  return newlink
}

function ensureLifelineHeights (e) {
  // iterate over all Activities (ignore Groups)
  var arr = myDiagram.model.nodeDataArray
  var max = -1
  for (let i = 0; i < arr.length; i++) {
    var act = arr[i]
    if (act.isGroup) continue
    max = Math.max(max, act.start + act.duration)
  }
  if (max > 0) {
    // now iterate over only Groups
    // eslint-disable-next-line no-redeclare
    for (let i = 0; i < arr.length; i++) {
      var gr = arr[i]
      if (!gr.isGroup) continue
      if (max > gr.duration) { // this only extends, never shrinks
        myDiagram.model.setDataProperty(gr, 'duration', max)
      }
    }
  }
  console.log(max)
}

// some parameters
var LinePrefix = 20 // vertical starting point in document for all Messages and Activations
var LineSuffix = 30 // vertical length beyond the last message time
var MessageSpacing = 20 // vertical distance between Messages at different steps
var ActivityWidth = 10 // width of each vertical activity bar
var ActivityStart = 5 // height before start message time
var ActivityEnd = 5 // height beyond end message time

export default {
  name: 'SquenceDiagram',
  props: {
    sData: {
      type: Object,
      default: () => {}
    }
  },
  data () {
    return {
      diagram: null,
      modelData: {
        class: 'go.GraphLinksModel'
      }
    }
  },
  watch: {
    sData: {
      handler: function (val) {
        const data = JSON.parse(JSON.stringify(val))
        Object.assign(this.modelData, data)
        if (this.diagram) {
          this.load()
        }
      },
      deep: true,
      immediate: true
    }
  },
  mounted () {
    this.init()
  },
  methods: {
    init () {
      this.diagram = $(go.Diagram, this.$el, {
        allowCopy: false,
        linkingTool: $(MessagingTool), // defined below
        'resizingTool.isGridSnapEnabled': true,
        draggingTool: $(MessageDraggingTool), // defined below
        'draggingTool.gridSnapCellSize': new go.Size(1, MessageSpacing / 4),
        'draggingTool.isGridSnapEnabled': true,
        // automatically extend Lifelines as Activities are moved or resized
        SelectionMoved: this.ensureLifelineHeights,
        PartResized: this.ensureLifelineHeights,
        'undoManager.isEnabled': true
      })
      // 定义竖直方向虚线（LifeLine）节点模板
      this.diagram.groupTemplate =
        $(go.Group, 'Vertical',
          {
            locationSpot: go.Spot.Bottom,
            locationObjectName: 'HEADER',
            minLocation: new go.Point(0, 0),
            maxLocation: new go.Point(9999, 0),
            selectionObjectName: 'HEADER'
          },
          new go.Binding('location', 'loc', go.Point.parse).makeTwoWay(go.Point.stringify),
          $(go.Panel, 'Auto',
            { name: 'HEADER' },
            $(go.Shape, 'Rectangle',
              {
                fill: $(go.Brush, 'Linear', { 0: 'green', 1: go.Brush.darkenBy('green', 0.1) }),
                stroke: null
              }),
            $(go.TextBlock,
              {
                margin: 5,
                font: '400 10pt Source Sans Pro, sans-serif',
                stroke: 'white'
              },
              new go.Binding('text', 'text'))
          ),
          $(go.Shape,
            {
              figure: 'LineV',
              fill: null,
              stroke: 'gray',
              strokeDashArray: [3, 3],
              width: 1,
              alignment: go.Spot.Center,
              portId: '',
              fromLinkable: true,
              fromLinkableDuplicates: true,
              toLinkable: true,
              toLinkableDuplicates: true,
              cursor: 'pointer'
            },
            new go.Binding('height', 'duration', (duration) => {
              return LinePrefix + duration * MessageSpacing + LineSuffix
            }))
        )

      // define the Activity Node template
      this.diagram.nodeTemplate =
        $(go.Node,
          {
            locationSpot: go.Spot.Top,
            locationObjectName: 'SHAPE',
            minLocation: new go.Point(NaN, LinePrefix - ActivityStart),
            maxLocation: new go.Point(NaN, 19999),
            selectionObjectName: 'SHAPE',
            resizable: true,
            resizeObjectName: 'SHAPE',
            resizeAdornmentTemplate:
              $(go.Adornment, 'Spot',
                $(go.Placeholder),
                $(go.Shape, // only a bottom resize handle
                  {
                    alignment: go.Spot.Bottom,
                    cursor: 'col-resize',
                    desiredSize: new go.Size(6, 6),
                    fill: 'yellow'
                  })
              )
          },
          new go.Binding('location', '', this.computeActivityLocation).makeTwoWay(this.backComputeActivityLocation),
          $(go.Shape, 'Rectangle',
            {
              name: 'SHAPE',
              fill: 'white',
              stroke: 'black',
              width: ActivityWidth,
              // allow Activities to be resized down to 1/4 of a time unit
              minSize: new go.Size(ActivityWidth, this.computeActivityHeight(0.25))
            },
            new go.Binding('height', 'duration', this.computeActivityHeight).makeTwoWay(this.backComputeActivityHeight))
        )

      // define the Message Link template.
      this.diagram.linkTemplate =
        $(MessageLink, // defined below
          { selectionAdorned: true, curviness: 0 },
          $(go.Shape, 'Rectangle',
            { stroke: 'black' }),
          $(go.Shape,
            { toArrow: 'OpenTriangle', stroke: 'black' }),
          $(go.TextBlock,
            {
              font: '400 9pt Source Sans Pro, sans-serif',
              segmentIndex: 0,
              segmentOffset: new go.Point(NaN, NaN),
              isMultiline: false,
              editable: true
            },
            new go.Binding('text', 'text').makeTwoWay())
        )

      myDiagram = this.diagram

      this.load()
    },
    computeActivityLocation (act) {
      var groupdata = this.diagram.model.findNodeDataForKey(act.group)
      if (groupdata === null) return new go.Point()
      // get location of Lifeline's starting point
      var grouploc = go.Point.parse(groupdata.loc)
      return new go.Point(grouploc.x, this.convertTimeToY(act.start) - ActivityStart)
    },
    // time is just an abstract small non-negative integer
    // here we map between an abstract time and a vertical position
    convertTimeToY (t) {
      return t * MessageSpacing + LinePrefix
    },
    convertYToTime (y) {
      return (y - LinePrefix) / MessageSpacing
    },
    backComputeActivityLocation (loc, act) {
      this.diagram.model.setDataProperty(act, 'start', this.convertYToTime(loc.y + ActivityStart))
    },
    computeActivityHeight (duration) {
      return ActivityStart + duration * MessageSpacing + ActivityEnd
    },
    backComputeActivityHeight (height) {
      return (height - ActivityStart - ActivityEnd) / MessageSpacing
    },
    ensureLifelineHeights (e) {
      // iterate over all Activities (ignore Groups)
      var arr = this.diagram.model.nodeDataArray
      var max = -1
      for (let i = 0; i < arr.length; i++) {
        var act = arr[i]
        if (act.isGroup) continue // 跳过组
        max = Math.max(max, act.start + act.duration)
      }
      console.log(max)
      if (max > 0) {
        // now iterate over only Groups
        // eslint-disable-next-line no-redeclare
        for (let i = 0; i < arr.length; i++) {
          var gr = arr[i]
          if (!gr.isGroup) continue // 跳过非租
          if (max > gr.duration) { // this only extends, never shrinks
            this.diagram.model.setDataProperty(gr, 'duration', max)
          }
        }
      }
      console.log(max)
    },
    load () {
      // this.diagram.model = go.Model.fromJson(document.getElementById('mySavedModel').value)
      this.diagram.model = go.Model.fromJson(this.modelData)
    },
    watch: {
      modelData: {
        handler: function (val) {
          this.ensureLifelineHeights()
        },
        deep: true
      }
    }
  }
}
</script>

<style>

</style>
