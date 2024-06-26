<template>
  <div class="_homeView">
    <div class="_sidebar">
      <h1>Cargo Bikes<br />Size Comparator</h1>

      <div class="_search">
        <input type="search" v-model="search_str" placeholder="Search by model or manufacturer" />
      </div>

      <div v-if="filtered_bikes.length === 0" class="_noMatch">
        No bikes matched your search.<br />
        To contribute a bike,
        <a href="https://github.com/louis-ev/Cargo-Bikes-Size-Comparator/issues/9" target="_blank"
          >read the guide</a
        >
        or ask me <a href="mailto:hello@louiseveillard.com" target="_blank">via email</a>.
      </div>

      <template v-else>
        <small class="_infos">
          <template v-if="enabled_bikes.length === 0">
            Click on bikes in this list to compare their size
          </template>
          <template v-else>
            <span>
              <template v-if="enabled_bikes.length === 1">
                {{ enabled_bikes.length }} bike selected
              </template>
              <template v-else> {{ enabled_bikes.length }} bikes selected </template>
            </span>
            <button class="_reset" @click="resetBikes">Reset</button>
          </template>
        </small>
      </template>

      <transition-group tag="div" class="_bikeList" name="list">
        <div
          class="_item"
          v-for="item in filtered_bikes"
          :key="item.id"
          :data-active="enabled_bikes.includes(item.id)"
        >
          <label :for="item.id" class="_itemTop">
            <input
              type="checkbox"
              :checked="enabled_bikes.includes(item.id)"
              :id="item.id"
              @change="toggleBike(item.id)"
            />

            <div class="_names">
              <strong
                >{{ item.model || item.manufacturer }}
                <span class="_flag" v-if="item.frame_made_in">
                  {{ unicodeFlag(item.frame_made_in) }}
                </span>
              </strong>
              <template v-if="item.manufacturer && item.model">
                <small> – {{ item.manufacturer }} </small>
              </template>
              <br />
              <small>{{ item.bike_length_cm }} cm</small>
            </div>

            <div v-if="item.id">
              <img :src="getBikeThumbImage(item)" />
            </div>
          </label>

          <div class="_itemBottom" v-if="enabled_bikes.includes(item.id)">
            <div class="_madeIn" v-if="item.frame_made_in">
              Bike mostly manufactured and assembled in <strong>{{ item.frame_made_in }}</strong
              >.
            </div>

            <div class="_measurements" v-if="item._measurements">
              <small v-html="getMeasurements(item)" />
              <br />
            </div>
            <div class="_source">
              <a :href="item.url" target="_blank"> <span>&#8594;</span>website</a>
            </div>
          </div>
        </div>
      </transition-group>

      <hr />
      <details>
        <summary>Advanced options</summary>
        <div class="_advanced">
          <label>Grid step (cm)</label>
          <input type="range" step="1" min="1" max="100" v-model.number="grid_step" />
        </div>
        <div class="_advanced">
          <label>Padding (%)</label>
          <input type="range" step="1" min="0" max="30" v-model.number="default_padding_percent" />
        </div>
        <div class="_advanced">
          <label>Layer blending</label>
          <select v-model="canvas_composite_operation">
            <option
              v-for="operation in globalCompositeOperations"
              :key="operation"
              :value="operation"
            >
              {{ operation }}
            </option>
          </select>
          <button @click="prevGlobalCompositeOperation">-</button>
          <button @click="nextGlobalCompositeOperation">+</button>
        </div>
      </details>

      <div class="_madeBy">
        <div>
          Created by <a href="https://louiseveillard.com/" target="_blank">Louis Eveillard</a> in
          Nantes (FR), with contributions from around the world.
        </div>
        <div>
          Report errors and bugs / send feedbacks / contribute bikes
          <a href="https://github.com/louis-ev/Cargo-Bikes-Size-Comparator" target="_blank"
            >on Github</a
          >
          or
          <a href="mailto:hello@louiseveillard.com" target="_blank">via email</a>
        </div>
        <div>
          No cookies, no tracking, no ads, and fully RGPD-compliant. Website hosted in France.
        </div>
        <hr />
        <div>
          Specific measures taken from
          <a
            href="https://docs.google.com/spreadsheets/d/1vPCfYStt8fXQQtYDFfNS70kR8B2V2dDwAs_r0VlUlWw/"
            target="_blank"
          >
            this document
          </a>
        </div>
        <div>
          Source code
          <a href="https://github.com/louis-ev/Cargo-Bikes-Size-Comparator" target="_blank">
            available on Github
          </a>
        </div>
        <div>
          Code/design
          <button type="button" class="noStyle" @click="show_license = !show_license">
            free, open-source under AGPLv3</button
          >, bike images from the manufacturer's website.
        </div>

        <template v-if="show_license">
          <div class="_license">
            <pre>
Copyright (C) 2024 Louis Eveillard

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License
along with this program.  If not, see http://www.gnu.org/licenses/.
              </pre
            >
          </div>
        </template>
      </div>
    </div>
    <div class="_canvasWrapper">
      <div class="_noBikes" v-if="enabled_bikes.length === 0">
        <small>Click on two bikes or more in the sidebar to compare their size</small>
      </div>
      <template v-else>
        <div class="_canvasOptions">
          <label class="_advanced _setImageStyle" for="canvas_image_style_outline">
            Outline View
            <input
              type="checkbox"
              name="canvas_image_style_outline"
              id="canvas_image_style_outline"
              v-model="canvas_image_style_outline"
            />
          </label>
        </div>
      </template>
      <canvas ref="bikes" width="1920" height="1920" />
      <canvas ref="processor" width="1920" height="1920" style="display: none" />
    </div>
  </div>
</template>
<script>
import { edge_detect, colorize } from '../helpers.js'
const bike_images_thumbs_paths = import.meta.glob('@/assets/bikes/*.png', {
  query: { format: 'png', w: 100 }
})
const bike_images_full_paths = import.meta.glob('@/assets/bikes/*.png', {
  query: { format: 'png' }
})

export default {
  props: {
    bikes: Array
  },
  components: {},
  data() {
    return {
      ro: null,

      default_padding_percent: 5,
      grid_step: 20,

      bike_images_thumbs_urls: [],
      bike_images_full_paths: [],

      search_str: '',
      show_license: false,

      canvas_composite_operation: 'source-over',
      globalCompositeOperations: [
        'source-over',
        'source-in',
        'source-out',
        'source-atop',
        'destination-over',
        'destination-in',
        'destination-out',
        'destination-atop',
        'lighter',
        'copy',
        'xor',
        'multiply',
        'screen',
        'overlay',
        'darken',
        'lighten',
        'color-dodge',
        'color-burn',
        'hard-light',
        'soft-light',
        'difference',
        'exclusion',
        'hue',
        'saturation',
        'color',
        'luminosity'
      ],

      canvas_image_style_outline: false
    }
  },
  created() {},
  async mounted() {
    this.bike_images_thumbs_urls = await this.loadAllThumbs()
    this.bike_images_full_paths = await this.loadAllFullPaths()

    this.showBikes()
    this.ro = new ResizeObserver(this.showBikes)
    if (this.$el) {
      this.ro.observe(this.$el)
    }
  },
  beforeUnmount() {
    this.ro.unobserve(this.$el)
  },
  watch: {
    enabled_bikes: {
      handler() {
        this.showBikes()
      },
      deep: true
    },
    canvas_composite_operation: {
      handler() {
        this.showBikes()
      }
    },
    canvas_image_style_outline: {
      handler() {
        // could add these defaults in if desired
        // if (this.canvas_image_style === 'line') {
        //   this.canvas_composite_operation = 'multiply'
        // } else {
        //   this.canvas_composite_operation = 'source-over'
        // }
        this.showBikes()
      }
    },
    default_padding_percent: {
      handler() {
        this.showBikes()
      }
    },
    grid_step: {
      handler() {
        this.showBikes()
      }
    }
  },
  computed: {
    enabled_bikes() {
      if (this.$route.query.bikes) {
        try {
          return JSON.parse(this.$route.query.bikes)
        } catch (e) {
          return []
        }
      }
      return []
    },
    filtered_bikes() {
      return this.bikes.filter((bike) => {
        return (
          this.normalizeStringForSearch(bike.model).includes(
            this.normalizeStringForSearch(this.search_str)
          ) ||
          this.normalizeStringForSearch(bike.manufacturer).includes(
            this.normalizeStringForSearch(this.search_str)
          )
        )
      })
    }
  },
  methods: {
    normalizeStringForSearch(str) {
      return str
        .toLowerCase()
        .normalize('NFD')
        .replace(/[\u0300-\u036f]/g, '')
    },

    findMatchingBike(id) {
      return this.bikes.find((i) => i.id === id)
    },
    resetBikes() {
      this.$router.push({
        query: {}
      })
    },
    unicodeFlag(country) {
      if (country === 'USA') return '🇺🇸'
      if (country === 'Germany') return '🇩🇪'
      if (country === 'France') return '🇫🇷'
      return
    },
    async loadAllThumbs() {
      const urls = []
      for (let [source, thumb] of Object.entries(bike_images_thumbs_paths)) {
        const import_statment = thumb()
        const url = (await import_statment).default
        const original_filename = source.split('/').pop()
        urls.push({
          url,
          original_filename
        })
      }
      return urls
    },
    async loadAllFullPaths() {
      const full_paths = []
      for (let [source, full_path] of Object.entries(bike_images_full_paths)) {
        const import_statment = full_path()
        const url = (await import_statment).default
        const original_filename = source.split('/').pop()
        full_paths.push({
          url,
          original_filename
        })
      }
      return full_paths
    },
    getBikeThumbImage(bike) {
      const thumb = this.bike_images_thumbs_urls.find((i) => i.original_filename === bike.src)
      if (!thumb) return
      return thumb.url
    },
    getBikeFullImage(bike) {
      const full_path = this.bike_images_full_paths.find((i) => i.original_filename === bike.src)
      if (!full_path) return
      return full_path.url
    },
    toggleBike(id) {
      let enabled_bikes = this.enabled_bikes
      if (enabled_bikes.includes(id)) {
        enabled_bikes = enabled_bikes.filter((i) => i !== id)
      } else {
        enabled_bikes.push(id)
      }
      this.$router.push({
        query: {
          bikes: JSON.stringify(enabled_bikes)
        }
      })
    },
    async showBikes() {
      const canvas = this.$refs.bikes
      if (!canvas) return

      canvas.width = Math.min(
        canvas.parentNode.clientWidth * window.devicePixelRatio,
        canvas.parentNode.clientHeight * window.devicePixelRatio * 1.5
      )
      canvas.height = Math.min(
        canvas.parentNode.clientHeight * window.devicePixelRatio,
        canvas.width
      )

      const ctx = canvas.getContext('2d')
      ctx.globalCompositeOperation = 'source-over'

      // bg
      ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue(
        '--color-background'
      )
      ctx.fillRect(0, 0, canvas.width, canvas.height)

      ctx.strokeStyle = '#ccc'
      ctx.strokeWidth = 1
      ctx.strokeRect(1, 1, canvas.width - 1, canvas.height - 1)

      // get largest bike image
      let largest_bike
      this.enabled_bikes.map((b) => {
        const bike = this.findMatchingBike(b)
        if (!largest_bike || largest_bike.bike_length_cm < bike.bike_length_cm) largest_bike = bike
      })

      if (!largest_bike) return

      const padding = canvas.width / (100 / this.default_padding_percent)
      const each_px_measures_in_cm = (canvas.width - padding * 2) / largest_bike.bike_length_cm

      ctx.strokeStyle = '#ccc'
      ctx.fillStyle = getComputedStyle(document.documentElement).getPropertyValue(
        '--color-text-secondary'
      )

      let cm_count = 0
      const step = this.grid_step

      for (let x = padding; x <= canvas.width; x += each_px_measures_in_cm * step) {
        ctx.beginPath()
        ctx.moveTo(x, 0)
        ctx.lineTo(x, canvas.height)
        ctx.stroke()

        const font_size = (canvas.parentNode.clientHeight / 80) * window.devicePixelRatio * 1
        ctx.font = `${font_size}px Inter`

        ctx.fillText(cm_count, x + 4, canvas.height - 4)
        cm_count += step
      }

      cm_count = 0
      for (let y = canvas.height - padding; y >= 0; y -= each_px_measures_in_cm * step) {
        ctx.beginPath()
        ctx.moveTo(0, y)
        ctx.lineTo(canvas.width, y)
        ctx.stroke()

        ctx.fillText(cm_count, 0 + 4, y - 4)

        cm_count += step
      }

      ctx.globalCompositeOperation = this.canvas_composite_operation

      const sorted_enabled_bikes = this.enabled_bikes
        .reduce((acc, id, index) => {
          const bike = this.findMatchingBike(id)
          if (!bike?.src) return

          // if (!bike.color) {
          const color_options = ['ff0000', '11bb11', '3333ff', 'bbbb00', 'ff00ff', '00bbbbb']
          bike.color = color_options[index % color_options.length]
          // }
          acc.push(bike)
          return acc
        }, [])
        .sort((a, b) => {
          return this.findMatchingBike(b)?.bike_length_cm - this.findMatchingBike(a)?.bike_length_cm
        })

      for await (const bike of sorted_enabled_bikes) {
        const img = new Image()

        img.src = this.getBikeFullImage(bike)
        await img.decode()

        const img_ratio = img.width / img.height
        const draw_w = (bike.bike_length_cm / bike.bike_length_percent) * each_px_measures_in_cm
        const draw_h = draw_w / img_ratio

        const draw_x = -bike.left_margin_percent * draw_w + padding

        const draw_y = canvas.height - padding - draw_h + bike.bottom_margin_percent * draw_h

        if (this.canvas_image_style_outline) {
          // Offscreen canvas for edge detection on the image
          const processorCanvas = this.$refs.processor
          if (!processorCanvas) return

          processorCanvas.width = Math.min(
            processorCanvas.parentNode.clientWidth * window.devicePixelRatio,
            processorCanvas.parentNode.clientHeight * window.devicePixelRatio * 1.5
          )
          processorCanvas.height = Math.min(
            processorCanvas.parentNode.clientHeight * window.devicePixelRatio,
            processorCanvas.width
          )

          const processorCtx = processorCanvas.getContext('2d')
          processorCtx.globalCompositeOperation = 'source-over'

          // white BG for a clean edge detect result
          processorCtx.fillStyle = 'white'
          processorCtx.fillRect(0, 0, canvas.width, canvas.height)
          processorCtx.drawImage(img, draw_x, draw_y, draw_w, draw_h)

          // detect edges
          edge_detect(processorCanvas)

          // colorize
          let color = bike.color
          colorize(processorCanvas, color)

          ctx.drawImage(processorCanvas, 0, 0, processorCanvas.width, processorCanvas.height)
        } else {
          ctx.drawImage(img, draw_x, draw_y, draw_w, draw_h)
        }
      }

      // repère
    },
    getMeasurements(bike) {
      return Object.entries(bike._measurements)
        .reduce((acc, [k, v]) => {
          if (k && k !== '\r' && v && v !== '\r') acc.push(`${k}: ${v}`)
          return acc
        }, [])
        .join('<br>')
    },
    prevGlobalCompositeOperation() {
      const index = this.globalCompositeOperations.indexOf(this.canvas_composite_operation)
      this.canvas_composite_operation = this.globalCompositeOperations[index - 1]
    },
    nextGlobalCompositeOperation() {
      const index = this.globalCompositeOperations.indexOf(this.canvas_composite_operation)
      this.canvas_composite_operation = this.globalCompositeOperations[index + 1]
    }
  }
}
</script>
<style lang="scss" scoped>
._homeView {
  display: flex;
  flex-flow: row nowrap;
  height: 100svh;
}

canvas {
  max-width: 100%;
  height: auto;
}

h1 {
  font-weight: 800;
}

._sidebar {
  flex: 0 0 auto;
  width: 320px;
  padding: 1rem;
  background-color: var(--color-background);

  overflow-y: auto;
}

._canvasWrapper {
  position: relative;
  flex: 1 1 auto;
  overflow: hidden;
  min-width: 420px;
  margin: 1rem;
  margin-left: 0;
  // border: 1px solid #ccc;
}
._noBikes {
  display: flex;
  flex-flow: column nowrap;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
  padding: 1rem;

  // font-weight: 400;
  text-transform: lowercase;
  color: var(--color-text-secondary);
}

._noMatch {
}

._bikeList {
  display: flex;
  flex-flow: column nowrap;
  gap: 0.5rem;
  padding: 0.25rem 0;
}

._item {
  line-height: 1.2;
  background-color: white;
  border-radius: 0.5rem;
  overflow: hidden;

  transition: all 0.5s cubic-bezier(0.19, 1, 0.22, 1);

  &[data-active='true'] ._itemTop {
    background-color: var(--color-accent);
  }

  &:hover,
  &:focus-visible {
    // box-shadow: 0 0.05rem 0.2rem 0rem var(--color-accent);
  }
}

._itemTop {
  padding: 0.25rem 1rem;
  cursor: pointer;
  border-radius: 0.5rem;

  display: flex;
  flex-direction: row nowrap;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;

  input {
    cursor: pointer;
  }

  ._names {
    flex: 1 1 auto;
  }

  input {
    flex: 0 0 auto;
  }
  img {
    flex: 0 0 auto;
    height: 30px;
    width: 40px;
    object-fit: scale-down;
    object-position: right;
  }

  &:hover,
  &:focus-visible {
    background-color: var(--color-accent);
  }
}

._advanced {
  display: flex;
  flex-direction: row nowrap;
  justify-content: space-between;
  align-items: center;
  gap: 0.5rem;

  margin-bottom: 0.25rem;
}

._itemBottom {
  padding: 0.5rem 1rem 1rem;
  display: flex;
  flex-flow: column nowrap;
  gap: 0.5rem;
}
._measurements {
  margin-bottom: 0.5rem;
}

.list-move, /* apply transition to moving elements */
.list-enter-active,
.list-leave-active {
  position: relative;
  transition: all 0.25s cubic-bezier(0.19, 1, 0.22, 1);
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateX(0px);
}

.list-leave-active {
  position: absolute;
  z-index: -1;
}

._madeBy {
  margin-top: 1rem;
  font-size: 0.8rem;
  color: var(--color-text-secondary);

  > * {
    margin-bottom: 0.5rem;
  }
}

._reset {
  padding: 0 0.25rem;

  cursor: pointer;
}
._search {
  margin-bottom: 1rem;

  input {
    width: 100%;
  }
}

._canvasOptions {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  pointer-events: none;
  padding: 1rem;

  display: flex;
  flex-direction: row wrap;
  justify-content: flex-start;
  gap: 0.5rem;

  > * {
    pointer-events: auto;
    background-color: white;
    border-radius: 0.5rem;
    padding: 0.5rem;
  }
}

._infos {
  display: flex;
  flex-direction: row wrap;
  justify-content: space-between;
  align-items: center;
  gap: 0.5rem;
}

._flag {
  font-size: 0.6rem;
  margin-left: 0.15rem;
}
</style>
