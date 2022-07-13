Flourish
========

Flourish is an epicyclic music visualiser. It takes the core concept of
Fourier analysis, that any signal can be represented as a sum of cycles, and
uses it to construct a spiral made out of the relationships between the
frequencies in music.

If you just want to see it in action, here's
[Flourish](https://demos.samgentle.com/flourish). If you want to know more,
read on!

The code has three essential building blocks:

* Audio, which takes input from the microphone and transforms it into a
frequency spectrum using the Web Audio API
* Spiral, which turns that frequency spectrum into a spiral shape that we can plot
* Canvas, which plots the spiral on the screen


Audio
-----

The Audio code sets up an audio context, handles the getUserMedia dance, and
acts as a little wrapper for AnalyserNode so we can get FFTs out.

  ```js
  class Audio {
    constructor() {
      const context = new AudioContext({
        latencyHint: 'interactive',
        sampleRate: 48000,
      })
      const input = context.createGain()

      const analyser = context.createAnalyser()
      analyser.fftSize = CONFIG.fftSize
      analyser.smoothingTimeConstant = CONFIG.smoothing
      input.connect(analyser)

      const buffer = new Float32Array(analyser.frequencyBinCount)

      this.currentSource = null

      Object.assign(this, {context, input, analyser, buffer})
    }

    async listen() {
      this.context.resume()

      const media = await navigator.mediaDevices.getUserMedia({
        audio: {
          echoCancellation: false,
          noiseSuppression: false,
          autoGainControl: false,
          sampleRate: 48000,
          latency: 0
        }
      })

      const source = this.context.createMediaStreamSource(media)
      source.connect(this.input)
      this.currentSource = source

      return source
    }

    fft() {
      this.analyser.getFloatFrequencyData(this.buffer)
      return this.buffer
    }

    stop() {
      const source = this.currentSource
      source?.disconnect(this.input)
      source?.mediaStream?.getTracks().forEach(track => track.stop())

      this.currentSource = null
      this.context.suspend()
    }
  }
  ```

Spiral
------

The Spiral is created from our FFT data. The whole spectrum would allow us to
reconstruct the signal in perfect detail, but we just want the most important
bits. So, we take the frequency data and only use the top K values – those
with the greatest contribution to the final signal. We also apply some
thresholding, mostly for performance.

Web Audio gives us values in dB, so we undo that to get the unscaled power
values back. We also halve all the frequencies so that we get a one-sided spiral
instead of a symmetric figure.

For scaling, we also keep track of the decibel value of the loudest frequency.

  ```js
  class Spiral {
    constructor(data) {
      const top = topk(data, CONFIG.complexity, CONFIG.threshold)

      const {volmin, volmax} = CONFIG
      const db = top[0]?.v ?? volmin

      this.vol = clamp((db - volmin) / (volmax - volmin), 0, 1)

      const freqs = top.map(d => ({
        ratio: d.i / 2,
        power: 10**(d.v/20),
      }))

      this.freqs = freqs
    }
  ```

getPoint is a parametric equation that turns values from 0->1 into coordinates
along the spiral by adding up all the frequencies at that point. It returns
values in an arbitrary range (depending on `power`), so we scale it later.

  ```js
    getPoint(t) {
      const {freqs} = this

      let x = 0
      let y = 0

      for (let i = 0; i < freqs.length; i++) {
        const v = freqs[i].power
        const f = freqs[i].ratio

        const n = t * f
        x += fastsin(n) * v
        y += fastsin(n+0.25) * v
      }

      return {x, y}
    }
  ```

We need a dummy spiral to display until we have real data to work with.

  ```js
    static dummy() {
      const spiral = new this([])
      spiral.freqs = [
        {ratio: 3, power: 0.3},
        {ratio: 2.5, power: 0.6},
        {ratio: 16, power: 0.1},
      ]
      spiral.vol = 1

      return spiral
    }
  }
  ```


Canvas
------

Canvas handles the plumbing of the canvas element and gets called in
the main loop to actually draw the spiral.

The fundamental spiral shape is represented by `Spiral`, but it's `Canvas`'s
job to turn that into pixels on the screen.

  ```js
  class Canvas {
    constructor(el) {
      this.el = el
      this.ctx = el.getContext('2d', {alpha: false})

      this.smoothVol = 0

      this.resize()
    }

    resize() {
      const {el} = this

      this.width = el.width = el.offsetWidth * window.devicePixelRatio
      this.height = el.height = el.offsetHeight * window.devicePixelRatio

      this.clear()
    }

    clear() {
      const {ctx} = this

      ctx.fillStyle = 'white'
      ctx.strokeStyle = 'black'
      ctx.lineWidth = window.devicePixelRatio

      ctx.fillRect(0, 0, this.width, this.height)
    }
  ```

We want to smooth out the data so that the spiral slowly responds to changes
in loudness. We do this in the draw loop, but we don't want it to depend on
the frame rate, so we scale it by the time since the last frame.

  ```js
    getSmoothVol(vol, dt) {
      if (dt) {
        const frames = dt / (1000 / 60)
        this.smoothVol = mix(vol, this.smoothVol, CONFIG.volsmoothing**frames)
      }
      else {
        this.smoothVol = vol
      }
      return this.smoothVol
    }
  ```

`draw` handles iterating through the points of the spiral, scaling them, and
drawing them to the canvas.

We use `p0`, which will always be the largest point of the spiral, to scale
the spiral to a uniform size. We then offset it vertically so that it
"grows" up from the bottom of the screen.

We apply two scaling factors based on the audio's volume:

* `size` scales `getPoint`'s *output*, making the spiral larger or smaller.
* `extent` scales `getPoint`'s *input*, making the spiral shorter or longer.

Together, these give the visualisation its characteristic unfurling movement.

  ```js
    draw(spiral, dt) {
      const {ctx, width, height} = this

      this.clear()
      if (!spiral.freqs[0]) return

      const vol = this.getSmoothVol(spiral.vol, dt)
      const size = vol**CONFIG.sizepower
      const extent = vol**CONFIG.extentpower

      const p0 = spiral.getPoint(0)
      const r0 = Math.max(p0.x, p0.y)
      const r = Math.min(width, height)/2

      const N = CONFIG.numpoints

      ctx.beginPath()
      for (let i = 0; i <= N; i++) {
        let {x, y} = spiral.getPoint(i/N * extent)

        x = size * x * r / r0 + width/2
        y = size * y * r / r0 + height - size * r

        ctx.lineTo(x, y)
      }
      ctx.stroke()
    }
  }
  ```


UI & Main loop
--------------

With all the major pieces in place, we can now proceed to the important task of doing frontend to it.

This basically involves reacting to various DOM events and plumbing those events through into the rest of the code.

  ```js
  async function main() {
    const canvas = new Canvas($('#canvas'))
    window.addEventListener('resize', canvas.resize.bind(canvas), {passive: true})
  ```

We draw a dummy spiral, and make sure it redraws on resize.

  ```js
    const drawDummy = () => canvas.draw(Spiral.dummy())
    drawDummy()

    window.addEventListener('resize', drawDummy, {passive: true})
  ```

Microphone access requires user input, so we wait until the splash screen is clicked.

We might not get access, in which case just chill. Maybe they'll change their mind?

  ```js
    const audio = new Audio()
    while (true) {
      try {
        await nextEvent(document, 'click')
        await audio.listen()
        break
      }
      catch {}
    }

    window.removeEventListener('resize', drawDummy)
    $('#splash').hidden = true
  ```

We give up microphone access when our page isn't visible. There's no point
running a visualisation in the background, and the "page is listening to
you" notification is creepy.

  ```js
    document.addEventListener('visibilitychange', () => {
      document.hidden ? audio.stop() : audio.listen()
    })
  ```

With everything in place, all that's left is to run our animation loop until
the end of time.

  ```js
    let lastt = performance.now()
    while (true) {
      const t = await nextFrame()

      const spiral = new Spiral(audio.fft())
      canvas.draw(spiral, t-lastt)

      lastt = t
    }
  }
  ```


Helpers
-------

Some GLSL-inspired math functions.

  ```js
  const clamp = (v, min, max) => Math.max(min, Math.min(max, v))
  const mix = (v0, v1, t) => v0 * (1-t) + v1 * t
  ```

Useful DOM functions.

  ```js
  const $ = document.querySelector.bind(document)

  const nextFrame = () => new Promise(requestAnimationFrame)
  const nextEvent = (el, name) =>
    new Promise(resolve =>
      el.addEventListener(name, resolve, {once: true, passive: true}))
  ```

A fast sine function using a lookup table, close enough for our purposes. It
takes input in turns, the One True Angular Unit.

  ```js
  const LUTN = 512
  const LUTMASK = LUTN-1
  const LUT = new Float32Array(LUTN)
  for (let i = 0; i < LUTN; i++) {
    LUT[i] = Math.sin(i/LUTN * Math.PI * 2)
  }
  const fastsin = (x) => {
    const i = x * LUTN
    const i0 = Math.floor(i)
    const i1 = Math.ceil(i)
    return mix(LUT[i0 & LUTMASK], LUT[i1 & LUTMASK], i-i0)
  }
  ```

A top-k algorithm that's not going to win many points on Leetcode, but does
win the "not having to implement a heap" prize for judicious use of time and
energy.

  ```js
  function topk(input, k, threshold) {
    const result = []

    let min = -Infinity

    for (let i = 0; i < input.length; i++) {
      const v = input[i]

      if (v < threshold || v < min) continue

      let newVal = {i, v}

      // We're holding a value, let's find a place to put it!
      for (let j = 0; j < k; j++) {
        let oldVal = result[j]

        // If we find an empty spot before the end, put the value there
        // (this only happens when the list is smaller than K)
        if (!oldVal) {
          result[j] = newVal
          break
        }

        // If we find a smaller value, replace it with our value
        // But now we need to find a place to put the value we replaced
        // So we pick it up instead and keep going
        if (newVal.v > oldVal.v) {
          result[j] = newVal
          newVal = oldVal

          if (j === k-1) min = v
        }
      }
    }

    return result
  }
  ```


Config
------

These are our tuneables, earned by undertaking the legendary trials of error.

  ```js
  const CONFIG = {
    fftSize: 2048,
    threshold: -90,
    complexity: 96,
    smoothing: 0.9,
    volmin: -80,
    volmax: -40,
    volsmoothing: 0.8,
    sizepower: 0.5,
    extentpower: 1.5,
    numpoints: 360*4,
  }
  ```


Hello, world
------------

We made it! Let's get started.

  ```js
  main()
  ```
