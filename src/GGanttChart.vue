<template>
  <div 
    id="g-gantt-chart"
    :style="{width: width, background: themeColors.background}"
  >
    <g-gantt-timeaxis
      v-if="!hideTimeaxis"
      :chart-start="chartStart"
      :chart-end="chartEnd"
      :row-label-width="rowLabelWidth"
      :timemarker-offset="timemarkerOffset"
      :theme-colors="themeColors"
      :locale="locale"
    />

    <g-gantt-grid 
      v-if="grid"
      :chart-start="chartStart"
      :chart-end="chartEnd"
      :row-label-width="rowLabelWidth"
      :highlighted-hours="highlightedHours"
    />
    
    <div id="g-gantt-rows-container">
      <slot/>   <!-- the g-gantt-row components go here -->
    </div>
    
  </div>
</template>

<script>
import moment from 'moment'
import GanttasticThemeColors from './GanttasticThemeColors.js'
import GGanttTimeaxis from './GGanttTimeaxis.vue'
import GGanttGrid from './GGanttGrid.vue'
import GGanttRow from './GGanttRow.vue'
import GGanttBar from './GGanttBar.vue'

export default {

  name: "GGanttChart",

  components:{
    GGanttTimeaxis,
    GGanttGrid
  },

  props:{
    chartStart: {type: String, default: moment().startOf("day").format("YYYY-MM-DD HH:mm:ss")},
    chartEnd: {type: String, default: moment().startOf("day").add(12,"hours").format("YYYY-MM-DD HH:mm:ss")},
    hideTimeaxis: Boolean,
    rowLabelWidth: {type: String, default: "10%"},
    rowHeight: {type: Number, default: 40},
    locale: {type: String, default: "en"},
    theme: {type: String},
    grid: {type: Boolean},
    highlightedHours: {type: Array, default: () => []},
    width: {type: String, default: "100%"},   // the total width of the entire ganttastic component in %
    pushOnOverlap: {type: Boolean},
    snapBackOnOverlap: {type: Boolean},
    minGapBetweenBars: {
      type: Number,
      default: 0
    }
  },

  data(){
    return{
      timemarkerOffset: 0,
      movedBarsInDrag: new Set()
    }
  },

  computed:{

    hourCount(){
      let momentChartStart = moment(this.chartStart)
      let momentChartEnd = moment(this.chartEnd)
      return Math.floor(momentChartEnd.diff(momentChartStart, "hour", true))
    },

    themeColors(){
      return GanttasticThemeColors[this.theme] || GanttasticThemeColors.default
    },

  },

  methods: {

    getGanttBarChildrenList(){
      let ganttBarChildren = []
      let ganttRowChildrenList = this.$children.filter(childComp => childComp.$options.name === GGanttRow.name)
      ganttRowChildrenList.forEach(row => {
        let ganttBarChildrenOfRow = row.$children.filter(childComp => childComp.$options.name === GGanttBar.name)
        ganttBarChildren.push(...ganttBarChildrenOfRow)
      })
      return ganttBarChildren
    },

    getBarsFromBundle(bundleId){
      if(bundleId === undefined || bundleId === null){
        return []
      }
      return this.getGanttBarChildrenList().filter(ganttBarChild => ganttBarChild.barConfig.bundle === bundleId)
    },

    initDragOfBarsFromBundle(gGanttBar, e){
      gGanttBar.initDrag(e)
      this.movedBarsInDrag.add(gGanttBar.bar)
      if(gGanttBar.barConfig.bundle !== null && gGanttBar.barConfig.bundle !== undefined){
        this.getGanttBarChildrenList().forEach(ganttBarChild => {
          if(ganttBarChild.barConfig.bundle === gGanttBar.barConfig.bundle && ganttBarChild !== gGanttBar){
            ganttBarChild.initDrag(e)
            this.movedBarsInDrag.add(ganttBarChild.bar)
          }
        })
      }
    },

    moveBarsFromBundleOfPushedBar(pushedBar, minuteDiff, overlapType){
      this.movedBarsInDrag.add(pushedBar)
      let bundleId = pushedBar.ganttBarConfig.bundle
      if(bundleId === undefined || bundleId === null){
        return
      }
      this.getGanttBarChildrenList().forEach(ganttBarChild => {
        if(ganttBarChild.barConfig.bundle === bundleId && ganttBarChild.bar !== pushedBar){
          ganttBarChild.moveBarByMinutesAndPush(minuteDiff, overlapType)
          this.movedBarsInDrag.add(ganttBarChild.bar)
        }
      })
    },

    shouldSnapBackBar(ganttBar){
      if(this.snapBackOnOverlap && ganttBar.barConfig.pushOnOverlap !== false){
        let {overlapBar} = ganttBar.getOverlapBarAndType(ganttBar.bar)
        return !!overlapBar
      }
      return false
    },

    snapBackBundleIfNeeded(ganttBar){
      let barsFromBundle = this.getBarsFromBundle(ganttBar.barConfig.bundle)
      if(this.shouldSnapBackBar(ganttBar) || barsFromBundle.some(gBar => this.shouldSnapBackBar(gBar))){
        ganttBar.snapBack()
        barsFromBundle.forEach(gBar => gBar.snapBack())
        return true
      }
      return false
    },

    onBarEvent(e, ganttBar){
      this.$emit(`${e.type}-bar`, {event: e, bar: ganttBar.bar})
    },
    
    onDragendBar(e, ganttBar){
      let didSnapBack = this.snapBackBundleIfNeeded(ganttBar)
      let movedBars = didSnapBack ? new Set() : this.movedBarsInDrag
      this.movedBarsInDrag = new Set()
      this.$emit("dragend-bar", {event: e, bar: ganttBar.bar, movedBars})
    },

    // ------------------------------------------------------------------------
    // --------  METHODS FOR SETTING THE DRAG LIMIT OF A BAR   ----------------
    // ------------------------------------------------------------------------

    // how far you can drag a bar depends on the position of the closest immobile bar
    // note that if a bar from the same row belongs to a bundle
    // other rows might need to be taken into consideration, too
    setDragLimitsOfGanttBar(bar){
      if(!this.pushOnOverlap || bar.barConfig.pushOnOverlap === false){
        return
      }
      for(let side of ["left", "right"]){
        let [totalGapDistance, bundleBarsOnPath] = this.countGapDistanceToNextImmobileBar(bar, null, side, false)
        for(let i=0; i< bundleBarsOnPath.length; i++){
          let barFromBundle = bundleBarsOnPath[i].bar
          let gapDist = bundleBarsOnPath[i].gapDistance
          let otherBarsFromBundle = this.getBarsFromBundle(barFromBundle.barConfig.bundle).filter(otherBar => otherBar !== barFromBundle)
          otherBarsFromBundle.forEach(otherBar => {
            let [newGapDistance, newBundleBars] = this.countGapDistanceToNextImmobileBar(otherBar, gapDist, side)
            if(newGapDistance !== null && (newGapDistance < totalGapDistance || !totalGapDistance)){
              totalGapDistance = newGapDistance
            }
            newBundleBars.forEach(newBundleBar => {
              if(!bundleBarsOnPath.find(barAndGap => barAndGap.bar === newBundleBar.bar)){
                bundleBarsOnPath.push(newBundleBar)
              }
            })
          })
        }
        if(totalGapDistance != null && side === "left"){
          bar.dragLimitLeft = bar.$refs['g-gantt-bar'].offsetLeft -  totalGapDistance
        } else if(totalGapDistance != null && side === "right"){
          bar.dragLimitRight = bar.$refs['g-gantt-bar'].offsetLeft+ bar.$refs['g-gantt-bar'].offsetWidth + totalGapDistance
        }
      }
      // all bars from the bundle of the clicked bar need to have the same drag limit:
      let barsFromBundleOfClickedBar = this.getBarsFromBundle(bar.barConfig.bundle)
      barsFromBundleOfClickedBar.forEach(barFromBundle => {
        barFromBundle.dragLimitLeft = bar.dragLimitLeft 
        barFromBundle.dragLimitRight = bar.dragLimitRight
      })
    },
    
    // returns the gap distance to the next immobile bar
    // in the row where the given bar (parameter) is (added to gapDistanceSoFar)
    // and a list of all bars on that path that belong to a bundle
    countGapDistanceToNextImmobileBar(bar, gapDistanceSoFar, side="left", ignoreShadows = true){
      let bundleBarsAndGapDist = bar.barConfig.bundle ? [{bar, gapDistance: gapDistanceSoFar}] : []
      let currentBar = bar
      let nextBar = this.getNextGanttBar(currentBar, side)
      // left side:
      if(side === "left"){
        while(nextBar){
          let nextBarOffsetRight = nextBar.$refs['g-gantt-bar'].offsetLeft + nextBar.$refs['g-gantt-bar'].offsetWidth
          gapDistanceSoFar += currentBar.$refs['g-gantt-bar'].offsetLeft - nextBarOffsetRight
          if(nextBar.barConfig.immobile || (nextBar.barConfig.isShadow && !ignoreShadows)){
            return [gapDistanceSoFar, bundleBarsAndGapDist]
          } else if(nextBar.barConfig.bundle){
            bundleBarsAndGapDist.push({bar: nextBar, gapDistance: gapDistanceSoFar})
          }
          currentBar = nextBar
          nextBar = this.getNextGanttBar(nextBar, "left")
        }
      }
      if(side === "right"){
        while(nextBar){
          let currentBarOffsetRight = currentBar.$refs['g-gantt-bar'].offsetLeft + currentBar.$refs['g-gantt-bar'].offsetWidth
          gapDistanceSoFar += nextBar.$refs['g-gantt-bar'].offsetLeft - currentBarOffsetRight
          if(nextBar.barConfig.immobile || (nextBar.barConfig.isShadow && !ignoreShadows)){
            return [gapDistanceSoFar, bundleBarsAndGapDist]
          } else if(nextBar.barConfig.bundle){
            bundleBarsAndGapDist.push({bar: nextBar, gapDistance: gapDistanceSoFar})
          }
          currentBar = nextBar
          nextBar = this.getNextGanttBar(nextBar, "right")
        }
      }
      return [null, bundleBarsAndGapDist]
    },

    getNextGanttBar(bar, side="left"){
      let allBarsLeftOrRight = []
      if(side === "left"){
        allBarsLeftOrRight = bar.$parent.$children.filter(gBar => {
          return gBar.$options.name === GGanttBar.name 
                  && gBar.$parent === bar.$parent 
                  && gBar.$refs['g-gantt-bar']
                  && gBar.$refs['g-gantt-bar'].offsetLeft < bar.$refs['g-gantt-bar'].offsetLeft
                  && gBar.barConfig.pushOnOverlap !== false
        })
      } else {
        allBarsLeftOrRight = bar.$parent.$children.filter(gBar => {
          return gBar.$options.name === GGanttBar.name 
                  && gBar.$parent === bar.$parent 
                  && gBar.$refs['g-gantt-bar']
                  && gBar.$refs['g-gantt-bar'].offsetLeft > bar.$refs['g-gantt-bar'].offsetLeft
                  && gBar.barConfig.pushOnOverlap !== false
        })
      }
      if(allBarsLeftOrRight.length > 0){
        return allBarsLeftOrRight.reduce(
          (bar1, bar2) => {
            let bar1Dist = Math.abs(bar1.$refs['g-gantt-bar'].offsetLeft - bar.$refs['g-gantt-bar'].offsetLeft)
            let bar2Dist = Math.abs(bar2.$refs['g-gantt-bar'].offsetLeft - bar.$refs['g-gantt-bar'].offsetLeft)
            return bar1Dist < bar2Dist ? bar1 : bar2
          },
          allBarsLeftOrRight[0]
        )
      } else {
        return null
      }
    }

    // ------------------------------------------------------------------------
    // ------------------------------------------------------------------------
    // ------------------------------------------------------------------------

  },
  
  // all child components of GGanttChart may have access to
  // the following values by using Vue's "inject" option:
  provide(){
    return {
      getChartStart: () => this.chartStart,
      getChartEnd: () => this.chartEnd,
      getHourCount: () => this.hourCount,
      ganttChartProps: this.$props,
      getThemeColors: () => this.themeColors,
      initDragOfBarsFromBundle: (bundleId, e) => this.initDragOfBarsFromBundle(bundleId, e),
      moveBarsFromBundleOfPushedBar: (bar, minuteDiff, overlapType) => this.moveBarsFromBundleOfPushedBar(bar, minuteDiff, overlapType),
      setDragLimitsOfGanttBar : (ganttBar) => this.setDragLimitsOfGanttBar(ganttBar),
      onBarEvent: (e, ganttBar) => this.onBarEvent(e, ganttBar),
      onDragendBar: (e, ganttBar) => this.onDragendBar(e, ganttBar),
      shouldSnapBackOnOverlap: () => this.snapBackOnOverlap,
      snapBackBundle: (ganttBar) => this.snapBackBundle(ganttBar),
      getMinGapBetweenBars: () => this.minGapBetweenBars
    }
  }
}
</script>

<style scoped>
  #g-gantt-chart{
    position: relative;
    display: flex;
    flex-direction: column;
    overflow-x: hidden;
    -webkit-touch-callout: none;
    -webkit-user-select: none;
    -khtml-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
    padding-bottom: 23px;
  }

  #g-gantt-chart >>> * {
    font-family: Roboto, Verdana;
  }

  #g-gantt-rows-container{
    position: relative;
  }
</style>