<template>
  <div>
    <div>
      <button @click="runTrackerInstance"> Run Task </button>
      <button @click="clearTimeline">Clear Timeline</button>
      <button
        v-if="canCancelLast && slowTask.lastCalled && !slowTask.lastCalled.isFinished"
        @click="slowTask.lastStarted.cancel()">
        Cancel Last
      </button>
      <button
        v-if="canCancelAll && slowTask.isActive"
        @click="slowTask.abort()">
        Cancel All
      </button>
    </div>

    <svg class="concurrencyTimeline">
      <g>
        <line
          :x1="xTimeline + '%'"
          y1="0"
          :x2="xTimeline + '%'"
          y2="100"
          stroke="black" />
      </g>
      <template
          v-for="instance in trackedInstances"
          v-if="instance.hasStarted">
        <g class="taskTrackerInstance">
          <rect
            :x="instance.g.x + '%'"
            :y="instance.g.y"
            :width="instance.g.width + '%'"
            :height="instance.g.height"
            :fill="instance.g.color"
            :fill-opacity="instance.g.opacity"
            stroke="black" />
          <text
            :x="instance.g.x + 0.5 + '%'"
            :y="instance.g.y + 15"
            :fill="instance.g.color"
            :text-decoration="[instance.taskInstance.isCanceled ? 'line-through' : 'none']"
            :font-style="[instance.startTime ? 'normal' : 'italic']"
            font-family="Raleway" font-size="14">
            {{ instance.taskInstance.state }}
          </text>
        </g>
      </template>
    </svg>
  </div>
</template>

<script>
/**
 * Enhance task property so that I can be used for tracker graph.
 */
function createTrackerInstance (host) {
  var colors = ['red', 'green', 'blue'],
      labelHeights = [0, 20, 40, 60, 80, 100]

  return {
    id: host.nextId++,
    calledTime: host.timeElapsed,
    hasStarted: false, // has its own start flag so that dropped tasks are shown
    startTime: false,
    endTime: false,
    taskInstance: false,

    g: {
      height: 20,
      width: 0.01,
      x: 0,
      y: pickFrom(labelHeights, host.nextId),
      color: pickFrom(colors, host.nextId),
      opacity: 0.3
    },

    update (currTime, upper, lower) {
      const ti = this.taskInstance

      if (!this.hasStarted) {
        this.startTime = currTime
        this.hasStarted = true
      }

      this.g.x = scale(this.startTime - lower, upper, lower)

      if (ti.isRunning) {
        this.g.width = scale(width(this, upper), upper, lower)
        this.endTime = currTime
      }
    }
  }
}

export default {
  props: {
    flow: {
      type: String, required: true
    },
    maxRunning: {
      type: Number, default: 1
    },
    delay: {
      type: Number, default: 0
    },
    canCancelLast: {
      type: Boolean, default: false
    },
    canCancelAll: {
      type: Boolean, default: false
    }
  },

  data () {
    return {
      trackedInstances: [],
      nextId: 0,
      startTime: false,
      timeElapsed: 0,
      xTimeline: 0
    }
  },
  tasks (t, { timeout }) {
    return {
      slowTask: t(function * () {
        yield timeout(2000)
      }).flow(this.flow, { delay: this.delay, maxRunning: this.maxRunning }),

      ticker: t(async (tracker) => {
        while (true) {
          this.timeElapsed = new Date() - this.startTime
          this.updateInstances()
          await timeout(10)
        }
      }).flow('drop')
    }
  },

  computed: {
    lowerLimit () {
      if (!this.trackedInstances) return 0
      const runTimes = []
      this.trackedInstances.forEach(instance => {
        runTimes.push(instance.calledTime)
      })
      return Math.min(...runTimes)
    },

    upperLimit () {
      return Math.max(10000, this.timeElapsed)
    }
  },

  methods: {
    runTrackerInstance () {
      this.startTime = this.startTime || new Date()

      const instance = createTrackerInstance(this)
      instance.taskInstance = this.slowTask.run()
      this.trackedInstances.push(instance)

      this.ticker.run(this.trackedInstances)
    },

    updateInstances () {
      var currTime = this.timeElapsed,
          upper = this.upperLimit,
          lower = this.lowerLimit

      window.requestAnimationFrame(() => {
        this.xTimeline = scale(currTime, upper, lower)
        this.trackedInstances.forEach(instance => {
          instance.update(currTime, upper, lower)
        })
      })
    },

    clearTimeline () {
      this.trackedInstances = []
      this.slowTask.abort()
      this.ticker.abort()
      this.nextId = 0
      this.startTime = 0
      this.timeElapsed = 0
      this.xTimeline = 0
    }
  }
}

function scale (value, upper, lower) {
  return Math.abs(100 * value / (upper + 1000 - lower))
}

function width (instance, upper) {
  var start = instance.startTime,
      end = instance.endTime
  return (end === Infinity) ? upper - start : end - start
}

function pickFrom (list, index) {
  return list[index % list.length]
}

</script>


<style lang="sass">
.concurrencyTimeline
  width: 100%
  padding: 1rem 0 0 .25rem
  // fix extra spacing
  margin-bottom: -2.5rem

.taskTrackerInstance
  height: 20px
</style>
