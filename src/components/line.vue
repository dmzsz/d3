<template>
  <div class="chart">
    <svg
      @mousemove="mouseover"
      @mouseleave="mouseleave"
      @click="clickLine"
      :width="width"
      :height="height"
      id="d3-svg"
    >
      <g id="x-axis" :style="{ transform: `translate(${0}px, ${padded.height + 10}px)` }" />
      <g
        id="y-axis"
        :style="{ transform: `translate(${margin.left}px, ${margin.top}px)` }"
        ref="yAxis"
      />
      <g :style="{ transform: `translate(${paddingOuter}px, ${margin.top}px)` }">
        <path class="area" :d="paths.area" />
        <template v-for="(line, index) in paths.lines">
          <path
            class="line"
            :d="line.path"
            :stroke="line.color"
            :key="index"
            fill="none"
            :stroke-width="
              paths.highlightLineId > -1 && index == paths.lines.length - 1
                ? currentLineStrokeWidth
                : lineStrokeWidth
            "
          />
        </template>
        <path class="selector" :d="paths.selector" />
      </g>
      <!-- <template v-for="(line, index) in animatedData">
        <g :style="{fill: line[0].color}" :key="index" :id="`points${index}`"></g>
      </template>-->
      <g class="points" :style="{ transform: `translate(${paddingOuter}px, ${margin.top}px)` }">
        <template v-for="linePoint in points">
          <template v-for="(point, pointIndex) in linePoint">
            <circle
              :cx="point.x"
              :cy="point.y"
              :r="currentPointRadius(point)"
              :fill="point.color"
              :key="`${point.id} +'_'+${pointIndex}`"
              @click="(e)=>clickPoint(e, point)"
            />
          </template>
        </template>
      </g>
    </svg>
    <div class="tooltip">
      <slot v-bind:default="lastHoverPoints">
      </slot>
    </div>
  </div>
</template>

<script>
import * as d3 from 'd3'

import TWEEN from 'tween.js'
import moment from 'moment'
export default {
  name: 'timeLineChart',
  props: {
    data: {
      type: Array,
      default: () => [100, 2, 13, 41, 5, 16, 71, 18, 19, 10]
    },
    margin: {
      type: Object,
      default: () => ({
        left: 30,
        right: 30,
        top: 10,
        bottom: 30
      })
    },
    ceil: {
      type: Number,
      default: 100
    }
  },
  data () {
    return {
      svg: null,
      yAxisWidth: 30,
      viewDates: [],
      width: 0,
      height: 200,
      paths: {
        area: '',
        lines: [],
        selector: '',
        highlightLineId: -1
      },
      lastHoverPoints: [],
      scaled: {
        x: null,
        y: null
      },
      animatedData: [],
      points: [],
      paddingOuter: 0,
      lineStrokeWidth: 1.5,
      currentLineStrokeWidth: 3
    }
  },
  computed: {
    padded () {
      const width = this.width - this.margin.left - this.margin.right
      const height = this.height - this.margin.top - this.margin.bottom
      return { width, height }
    }
  },

  watch: {
    data: function dataChanged (newData, oldData) {
      this.animatedData = newData
    },
    width: function widthChanged (data) {
      this.updateAxis()
      this.updateLine()
    },
    animatedData (val) {
      this.viewDates = this.animatedData
        .reduce((all, line) => {
          let lineDates = line.map(point => moment(point.date).valueOf())
          all = Array.from(new Set(all.concat(lineDates)))
          return all
        }, [])
        .sort((a, b) => {
          return a - b
        })

      this.updateAxis()
      this.updateLine()
    },
    'paths.highlightLineId' (val, oldVal) {
      if (oldVal > -1) {
        let oldIndex = this.paths.lines.findIndex(
          point => point._id === oldVal
        )
        this.paths.lines[oldIndex].strokeWidth = this.lineStrokeWidth
      }
      if (val > -1) {
        let index = this.paths.lines.findIndex(point => point._id === val)
        this.paths.lines.splice(
          this.paths.lines.length - 1,
          0,
          ...this.paths.lines.splice(index, 1)
        )
      }
      if (val > -1) {
        let index = this.points.findIndex(points => points[0]._id === val)
        this.points.splice(
          this.points.length - 1,
          0,
          ...this.points.splice(index, 1)
        )
      }
    },
    lastHoverPoints: {
      handler (val) {
        if (val.length > 0) {
        }
      }
    }
  },
  methods: {
    onResize () {
      this.width = this.$el.offsetWidth
      // this.height = this.$el.offsetHeight;
    },
    tweenData (newData, oldData) {
      // const vm = this
      function animate (time) {
        requestAnimationFrame(animate)
        TWEEN.update(time)
      }
      // new TWEEN.Tween(oldData)
      //   .easing(TWEEN.Easing.Quadratic.Out)
      //   .to(newData, 500)
      //   .onUpdate(function onUpdate() {
      //     vm.animatedData = this;
      //     vm.update();
      //   })
      //   .start();
      animate()
    },

    updateAxis () {
      if (this.padded.width < 0) return
      let labelWidth = 60 // x label Width 60px
      let allDate = d3.timeDay
        .range(
          this.viewDates[0],
          moment(this.viewDates.slice(-1)[0]).add(1, 'day')
        )
        .map(date => {
          return moment(date).toDate()
        })
      let maxAxisLabelNum = Math.floor(this.padded.width / labelWidth + 0) - 1
      let firstIndex = 0
      let scaleWidth = this.padded.width / (allDate.length + 1)
      if (scaleWidth < labelWidth / 2) {
        firstIndex = Math.ceil(labelWidth / 2 / scaleWidth) - 1
      }
      let stepWidth = Math.ceil(allDate.length / maxAxisLabelNum)
      let xAxisLabels = []
      if (maxAxisLabelNum < allDate.length) {
        let beforeIndex = firstIndex
        allDate.forEach((date, index) => {
          if (index === firstIndex || index === beforeIndex + stepWidth) {
            beforeIndex = index
            xAxisLabels.push(date)
          }
        })
      } else {
        xAxisLabels = allDate
      }
      let scaleBand = d3
        .scaleBand()
        .range([this.margin.left, this.padded.width + this.margin.left])
        .domain(allDate)
        .paddingOuter(0.5) // scaleWidth * 0.5 px
      this.paddingOuter = scaleWidth * 0.5

      this.scaled.x = scaleBand
      // this.scaled.x = d3
      //   .scaleLinear()
      //   .range([this.margin.left, this.padded.width + this.margin.left])
      //   .domain([this.viewDates[0], this.viewDates.slice(-1)])

      this.scaled.y = d3
        .scaleLinear()
        .range([this.padded.height, 0])
        .domain([
          this.minValue(this.animatedData),
          this.maxValue(this.animatedData)
        ])
        .nice()

      this.svg
        .selectAll('#x-axis')
        // .attr('transform', 'translate(30,' + (this.padded.height + 10) + ')')
        .call(
          d3
            .axisBottom(scaleBand)
            .tickFormat((date, i) => {
              if (
                i === 0 ||
                new Date(xAxisLabels[i - 1]).getFullYear() <
                  new Date(date).getFullYear()
              ) {
                return d3.timeFormat('%Y/%m/%d')(date)
              } else {
                return d3.timeFormat('%m/%d')(date)
              }
            })
            .tickValues(xAxisLabels)
        )
      this.svg
        .selectAll('#y-axis')
        // .attr('transform', 'translate(' + 30 + ',10)')
        .call(d3.axisLeft(this.scaled.y).ticks(3))
    },
    updateLine () {
      this.points = []
      for (const [lineIndex, lineData] of this.animatedData.entries()) {
        lineData.forEach((data, index) => {
          let findIndex = this.viewDates.findIndex(
            date => date === moment(data.date).valueOf()
          )
          if (findIndex > -1) {
            if (!this.points[lineIndex]) {
              this.points[lineIndex] = []
            }
            this.points[lineIndex].push(
              Object.assign(data, {
                x: this.scaled.x(moment(data.date).toDate()),
                y: this.scaled.y(data.value),
                max: this.height,
                color: data.color,
                _pointIndex: index,
                _id: lineIndex
              })
            )
          }
        })
      }

      // this.paths.area = this.createArea(this.points)
      this.points.forEach((line, index) => {
        if (!this.paths.lines[index]) {
          this.paths.lines[index] = {}
        }
        if (!this.paths.lines[index]) {
          this.paths.lines[index] = {}
        }
        this.paths.lines[index].path = this.createLine(line)
        this.paths.lines[index].strokeWidth = this.lineStrokeWidth
        this.paths.lines[index].color = line[0].color
        this.paths.lines[index]._id = line[0]._id
      })
    },
    clickLine ({ offsetX, offsetY }, index) {
      const x = offsetX - this.paddingOuter
      const y = offsetY - this.margin.top
      const closestPointRange = this.getClosestPointRange(x, y)
      let hasChangehighlight = false
      closestPointRange.forEach(line => {
        let height = this.pointHeight(line)
        if (height < 5) {
          // 5px
          this.paths.highlightLineId = line[0]._id
          hasChangehighlight = true
        }
      })
      if (!hasChangehighlight) this.paths.highlightLineId = -1
    },
    clickPoint (e, point) {},
    currentPointRadius (point) {
      if (
        this.lastHoverPoints.find(
          hoverPoint =>
            hoverPoint._id === point._id &&
            hoverPoint._pointIndex === point._pointIndex
        )
      ) {
        return 7
      } else {
        return 5
      }
    },
    mouseover ({ offsetX, offsetY }) {
      this.drawLine({ offsetX, offsetY })
    },
    mouseleave ({ offsetX, offsetY }) {
      // let alignmentLine = d3
      //   .select('#d3-svg')
      //   .selectAll('.alignment-line')
      //   .data([])
      // alignmentLine.exit().attr('fill', 'red').remove()
    },
    drawLine ({ offsetX, offsetY }) {
      const x = offsetX - this.paddingOuter
      const y = offsetY - this.margin.top
      let closestPoints = []
      if (y >= this.padded.height || y <= 0) {
        closestPoints = []
        this.hiddenTooltip()
      } else {
        closestPoints = this.getClosestPoints(x)
        this.showTooltip(closestPoints, { offsetX, offsetY })
      }
      if (
        (this.lastHoverPoints[0] !== undefined &&
          closestPoints[0] !== undefined &&
          this.lastHoverPoints[0]._pointIndex !==
            closestPoints[0]._pointIndex) ||
        (this.lastHoverPoints[0] !== undefined &&
          closestPoints[0] === undefined) ||
        (this.lastHoverPoints[0] === undefined &&
          closestPoints[0] !== undefined)
      ) {
        // const point = this.points[closestPoints.index]
        // this.paths.selector = this.createValueSelector([point]);closestPoints[0])

        this.$emit('select', closestPoints)
        this.lastHoverPoints = closestPoints || []
      }

      // let alignmentLine = d3.select('#d3-svg').select('.alignment-line')
      // let alignmentLine = d3
      //   .select('#d3-svg')
      //   .selectAll('.alignment-line')
      //   .data(closestPoints.length > 0 ? [0] : [])

      // alignmentLine
      //   .enter()
      //   .append('g')
      //   .attr('class', 'alignment-line')
      //   .attr(
      //     'transform',
      //     `translate(${this.paddingOuter}, ${this.margin.top})`
      //   )

      // alignmentLine.exit().remove()
      // let lineData = alignmentLine.data(closestPoints)

      let line = d3
        .select('#d3-svg')
        .selectAll('.alignment-line')
        .data(closestPoints.slice(0, 1))
      line
        .join('line')
        .attr('x1', d => d.x)
        .attr('x2', d => d.x)
        .attr('y1', d => this.padded.height)
        .attr('class', 'alignment-line')
        .style('stroke', '#0009')
        .attr(
          'transform',
          `translate(${this.paddingOuter}, ${this.margin.top})`
        )
      line.exit().remove()
    },
    showTooltip (closestPoints, { offsetX, offsetY }) {
      closestPoints.forEach(point => {
        point.marker = `<span style="display:inline-block;margin-right:5px;border-radius:10px;width:10px;height:10px;background-color:${point.color};"></span>`
        point.data = point.value
      })
      console.log(closestPoints)
      this.tooltipFormatter(closestPoints)
      let tooltip = document.querySelector('.chart .tooltip')
      tooltip.html()
    },
    tooltipFormatter (closestPoints, ticket, callback) {
      // let tooltip = document.querySelector('.chart .tooltip')
      // let res = moment(params[0].data[0]).format('L') + '</br>'
      // for (let i = 0; i < params.length; i++) {
      //   if (params[i].data[1] != undefined || params[i].data[1] != '') {
      //     res +=
      //       params[i].marker +
      //       params[i].seriesName +
      //       ':' +
      //       params[i].data[1] +
      //       '</br>'
      //   }
      // }
      // return res
    },
    hiddenTooltip () {},
    randomLetters () {
      return d3
        .shuffle('abcdefghijklmnopqrstuvwxyz'.split(''))
        .slice(0, Math.floor(6 + Math.random() * 20))
        .sort()
    },
    createArea: d3
      .area()
      .x(d => d.x)
      .y0(d => d.max)
      .y1(d => d.y),

    createLine: d3
      .line()
      .x(d => d.x)
      .y(d => d.y),

    createValueSelector: d3
      .area()
      .x(d => d.x)
      .y0(d => d.max)
      .y1(0),
    maxDate (lines) {
      let maxDate = ''
      lines.forEach((line, index) => {
        let lineMax = d3.max(line, d => moment(d.date).valueOf())
        if (lineMax > maxDate || index === 0) {
          maxDate = lineMax
        }
      })
      return maxDate
    },
    minDate (lines) {
      let minDate = ''
      lines.forEach((line, index) => {
        let lineMin = d3.min(line, d => moment(d.date).valueOf())
        if (lineMin < minDate || index === 0) {
          minDate = lineMin
        }
      })
      return minDate
    },
    maxValue (lines) {
      let maxValue = ''
      lines.forEach((line, index) => {
        let lineMax = d3.max(line, d => d.value)
        if (lineMax > maxValue || index === 0) {
          maxValue = lineMax
        }
      })
      return maxValue
    },
    minValue (lines) {
      let minValue = ''
      lines.forEach((line, index) => {
        let lineMin = d3.min(line, d => d.value)
        if (lineMin < minValue || index === 0) {
          minValue = lineMin
        }
      })
      return minValue
    },
    getClosestPointRange (x, y) {
      let adjacentXPoints = []
      this.points.forEach((linePoints, lineIndex) => {
        let diffX = linePoints.map((point, pointIndex) =>
          Object.assign(point, {
            _diffX: point.x - x
          })
        )
        if (diffX.length >= 2) {
          diffX.sort((a, b) => {
            return Math.abs(a._diffX) - Math.abs(b._diffX)
          })
          if (diffX[0]._diffX > 0) {
            let leftPoint = diffX.find(
              point => point._pointIndex === diffX[0]._pointIndex - 1
            )
            if (leftPoint) {
              adjacentXPoints.push([leftPoint, diffX[0]])
            }
          } else {
            let rightPoint = diffX.find(
              point => point._pointIndex === diffX[0]._pointIndex + 1
            )
            diffX[0]._diffX = Math.abs(diffX[0]._diffX)
            if (rightPoint) {
              adjacentXPoints.push([diffX[0], rightPoint])
            }
          }
        }
      })

      let intervalLines = []

      adjacentXPoints.forEach((line, lineIndex) => {
        line.forEach(point => {
          point.distance = Math.sqrt(
            Math.pow(point.x - x, 2) + Math.pow(point.y - y, 2)
          )
          point._id = point.id
        })
        intervalLines.push(line)
      })

      return intervalLines
    },

    pointHeight (line) {
      let bDistance = line[0].distance
      let cDistance = line[1].distance
      let aDistance = Math.sqrt(
        Math.pow(line[0].x - line[1].x, 2) + Math.pow(line[0].y - line[1].y, 2)
      )
      return (
        (1 / aDistance) *
        Math.sqrt(
          Math.pow(aDistance * bDistance, 2) -
            Math.pow(
              (Math.pow(aDistance, 2) +
                Math.pow(bDistance, 2) -
                Math.pow(cDistance, 2)) /
                2,
              2
            )
        )
      )
    },

    getClosestPoints (x) {
      let closestPoints = []
      this.points.forEach((linePoints, lineIndex) => {
        let closestPoint = linePoints
          .map((point, pointIndex) =>
            Object.assign(point, {
              _diffX: Math.abs(point.x - x)
            })
          )
          .reduce((memo, val) => (memo._diffX < val._diffX ? memo : val))
        if (closestPoint._diffX < this.scaled.x.step() / 2) {
          closestPoints.push(closestPoint)
        }
      })
      return closestPoints || []
    }
  },

  mounted () {
    this.svg = d3.select('#d3-svg')

    // this.animatedData = []
    // // let color = ['green', 'blue', 'yellow', 'orange', 'red'];
    // let color = ['#7cb55d', '#5d72b5', '#d9ca64', '#b5785d', '#5daeb5']
    // let name = [
    //   '生活リズム',
    //   '課題遂行率',
    //   '健康感',
    //   'コミュニケーション評価',
    //   '抑うつ・不安'
    // ];
    // [...new Array(5)].forEach((_, lineIndex) => {
    //   if (!this.animatedData[lineIndex]) this.animatedData[lineIndex] = [];
    //   [...new Array(3 * 7)].forEach((_, index) => {
    //     if (lineIndex === 0 || lineIndex === 4) {
    //       this.animatedData[lineIndex][this.animatedData[lineIndex].length] = {
    //         date: moment('2020-12-25')
    //           .startOf('day')
    //           .add(index, 'day')
    //           .format('YYYY-MM-DD'),
    //         value: Math.ceil(Math.random() * 100),
    //         color: color[lineIndex],
    //         id: lineIndex,
    //         seriesName: name[lineIndex]
    //       }
    //     }
    //     if (lineIndex === 1 && index % 7 < 5) {
    //       this.animatedData[lineIndex][this.animatedData[lineIndex].length] = {
    //         date: moment('2020-12-25')
    //           .startOf('day')
    //           .add(index, 'day')
    //           .format('YYYY-MM-DD'),
    //         value: Math.ceil(Math.random() * 100),
    //         color: color[lineIndex],
    //         id: lineIndex,
    //         seriesName: name[lineIndex]
    //       }
    //     }

    //     if (lineIndex === 2 && index % 7 < 2) {
    //       this.animatedData[lineIndex][this.animatedData[lineIndex].length] = {
    //         date: moment('2020-12-25')
    //           .startOf('day')
    //           .add(index, 'day')
    //           .format('YYYY-MM-DD'),
    //         value: Math.ceil(Math.random() * 100),
    //         color: color[lineIndex],
    //         id: lineIndex,
    //         seriesName: name[lineIndex]
    //       }
    //     }

    //     if (lineIndex === 3 && index % 7 < 1) {
    //       this.animatedData[lineIndex][this.animatedData[lineIndex].length] = {
    //         date: moment('2020-12-25')
    //           .startOf('day')
    //           .add(index, 'day')
    //           .format('YYYY-MM-DD'),
    //         value: Math.ceil(Math.random() * 100),
    //         color: color[lineIndex],
    //         id: lineIndex,
    //         seriesName: name[lineIndex]
    //       }
    //     }
    //   })
    // })

    this.updateAxis()
    window.addEventListener('resize', this.onResize)
    this.onResize()

    document.addEventListener('mouseleave', e => {
      let svg = document.getElementById('d3-svg')
      if (!svg.contains(e.target)) {
        this.lastHoverPoints = []
      }
    })

    document.addEventListener('click', e => {
      let svg = document.getElementById('d3-svg')
      if (!svg.contains(e.target)) {
        this.lastHoverPoints = []
      }
    })
  },
  beforeDestroy () {
    window.removeEventListener('resize', this.onResize)
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.chart {
  height: 100%;
  width: 100%;
  -webkit-tap-highlight-color: transparent;
  user-select: none;
  position: relative;
}
.tooltip {
  position: absolute;
}
</style>
