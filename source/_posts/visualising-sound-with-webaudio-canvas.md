---
title: Visualising sound with the WebAudio and Canvas APIs
tags:
  - audio
  - music
  - graphics
date: 2017-11-01 11:18:47
---


# Inspiration
I've always enjoyed the idea of turning audio into graphics, ever since the days of Windows Media Player's incredibly naff yet somewhat entertaining visualiser feature. A few years ago I discovered [raster-noton](), the great German record label which has been home to music from the likes of Robert Lippok, Aoki Takamasa and Ryoji Ikeda. More recently I came across this video for [atom™]()'s track, 'strom' on raster's YouTube channel.

<iframe width="560" height="315" src="https://www.youtube.com/embed/RpAl3ih5BmA" frameborder="0" allowfullscreen></iframe>

This waveform oscilloscope-like visualisation is quite cool. It's nothing hugely original, but it's nice and clean looking.

I then clicked on a recommended video to find this unofficial video for one of my favourite tracks, Robert Lippok's 'Whitesuperstructure'.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Pb42GbADwZM" frameborder="0" allowfullscreen></iframe>

Now this one's even *cooler*! It got me thinking of how hard it would be for me to write some software to generate these kind of visualisations in real time. Perhaps in, say, **JavaScript**...? ...And maybe make it into a web application.

# Tools
If we want to take some sound and turn it into visuals, we need some way of dealing with each. Whilst we could probably go in really low level here, for working on the Web, the [WebAudio]() and [HTML Canvas]() APIs will do a very good job.

# Setting up the basics

## Playing some audio
We've got a few options if we want to play audio, but let's just go with a simple `<audio>` tag in an HTML document for now.

```html
    <audio id="in" src="music.mp3" type="audio/mp3" autoplay></audio>
```

This should work straight away. When we load the page, we should hear the music begin to play.

## Getting ready to do graphics
Since we want a canvas for our output graphics, let's add a `<canvas>` tag.

```html
    <canvas id="out"></canvas>
```

How minimalist.

# Making a framework
We *could* achieve what we want simply by putting all of our logic in one place and intimately coupling our canvas and our audio with our code, but a much better practice is to follow the UNIX philosophy and write small, modular - and therefore reusable - components.

So let's make a `Visualiser` object/constructor/class/prototype/whatever. It's basically a class but since JavaScript uses a system of prototypal inheritance rather than the more common classical inheritance, I'm reluctant to call it a 'class'. It's basically a class, though. It's a function that returns an object instance when invoked using the `new` keyword.

## Constructing a Visualiser
To define what a `Visualiser` object is, we will write a constructor function.

The `Visualiser` will keep track of an audio source, and 'visualise' this out onto the canvas. It will use something I'll call a *render function* to do the actual drawing. More on this later.

First we'll call some methods (which we will define in a minute) to initialise stuff. Then we'll start calling the custom *render function*. We'll make a call to the standard `window.requestAnimationFrame` function and pass it an anonymous function as the callback. Within this anonymous function, we will call the custom `render function`, passing it a reference to the `Visualiser` instance that called it, and then make another call back to `window.requestAnimationFrame` to render the next frame. This cycle will repeat for the lifetime of the `Visualiser`. Delegating the responsibility for deciding when to render frames like this is good because it means that the browser can perform small optimisations such as pausing the canvas while the the containing tab is inactive.

There is one more property that we will set - `previousAmplitude`. This is, in my mind, a private property, although JavaScript doesn't *easily* support things. It *is* possible to use some other workarounds to get private members in objects but I'm just going to prepend an underscore `_` to its name to denote the privateness. The exact use of this particular property will be explained later...

```javascript
    function Visualiser(canvasID, audioID, renderFunction) {
        this.initialiseCanvas(canvasID);
        this.initialiseAudio(audioID);
        this.initialiseNodes();

        // Private properties
        this._previousAmplitude = 0;

        // Get references to visualiser instance and render function
        const v = this;
        const render = renderFunction;

        // Request animation frame using custom render function
        window.requestAnimationFrame(function renderFrame() {
            render(v);
            window.requestAnimationFrame(renderFrame);
        });
    }
```

## Initialising a Visualiser

### The canvas
Now that we've got our basic constructor function for `Visualiser` objects, let's implement those initialisation methods we made calls to.

We'll add these methods to the prototype of the `Visualiser` constructor function. This means that every `Visualiser` instance will refer back to the same initialisation function which is slighty more efficient than each instance containing its own copy of the function. We'll probably only ever have one `Visualiser`, but who knows! Always make room for the future.

To prepare the canvas, we need to get it from the DOM, and then set its properties to the values we want. The `Visualiser` already knows the `id` of the canvas element that it's using as input, so we can just use `document.querySelector` to retrieve a reference to it.

We'll also get the `pixelRatio` property on the `window` object and use it to scale the canvas so that it looks good on HiDPI displays because HTML Canvas looks ugly by default on modern displays.

Apart from setting properties on the canvas and configuring the canvas' styles, we'll also get a `RenderingContext2D` that we can use to directly draw to the canvas. We'll bind this as a property called `renderingContext` on the object so we can access it later.

```javascript
    Visualiser.prototype.initialiseCanvas = function(id) {
        // Create canvas rendering context
        this.canvas = document.querySelector(`#${id}`);
        this.renderingContext = this.canvas.getContext('2d');

        // Make canvas fill screen
        this.canvas.width = document.body.clientWidth;
        this.canvas.height = document.body.clientHeight;
        
        // Fix resolution
        const pixelRatio = window.devicePixelRatio;
        this.canvas.width *= pixelRatio;
        this.canvas.height *= pixelRatio;
        this.canvas.style.width = this.canvas.width/pixelRatio + "px";
        this.canvas.style.height = this.canvas.height/pixelRatio + "px";
    };
```

### The audio
Next, we need to initialise our audio within the `Visualiser`. The WebAudio API represents all the audio processing as a graph. Basically, we've got a bunch of nodes that can be connected to each other in various ways. It's quite cool.

The first thing we need is an `AudioContext` instance. Everything happens within, or in conjunction with, this object. We create create one using a method on the `window` and make this a property on the `Visualiser`'s prototype, and we set another property to reference the audio element in the same way as we did with the canvas rendering context.

```javascript
    Visualiser.prototype.initialiseAudio = function(id) {
        // Create audio context and load audio
        const Context = window.AudioContext || window.webkitAudioContext;
        this.audioContext = new Context;
        this.audioElement = document.querySelector(`#${id}`);
    };
```

### The nodes
We've got a context, so now we need some actual nodes. We use a `MediaElementSourceNode` to represent our audio source, and we create it by passing in our existing `<audio>` element from earlier. This node will act as our ultimate audio source within the graph.

WebAudio allows us to do all sorts of fancy processing and even signal generation (with oscillators) but what we want to do is extract data from existing audio. The `AnalyserNode` is perfect for this. We can create one using another property on the `window`, `createAnalyser`.

In our audio graph, our nodes need to be connected. Intuitively, our source node should be connected to the analyser node so that we can extract data, but we'll also connect the output of the analyser to the `destination` property of the audio context. Connecting a node to the `destination` allows us to actually hear the audio as well as get data from it!

```javascript
    Visualiser.prototype.initialiseNodes = function() {
        // Create nodes
        const source = this.audioContext.createMediaElementSource(this.audioElement);

        this.analyser = this.audioContext.createAnalyser();
        this.analyser.fftSize = 2048; // (default fft size)

        const destination = this.audioContext.destination;

        // Connect nodes
        source.connect(this.analyser);
        this.analyser.connect(destination);
    };
```

## Getting some actual data

### The amplitude
When we're visualising audio, probably the most important data value we want as input is the amplitude of the signal. Let's add a method for getting that value.

We can use the `getByteTimeDomainData` method on the analyser node to retrieve an array of bytes, with each byte corresponding to the displacement of the waveform at a given time. The time interval between these measurements is determined by the `fftSize` property that we set earlier, but we actually don't care about it that much because we're not going to use the points individually. As long as they collectively represent a fairly short time period (a few milliseconds), it's fine.

In order to get an amplitude that makes sense, we'll find the maximum value of the data we have retrieved. This gives us the maximum displacement of the wave which is the amplitude within the period. To make the data we're returning more usable, we'll also scale it to between 0 and 1 using a little bit of maths.

The `maths.max` function is one that I wrote separately to make this easier. You should be able to implement that yourself fairly easily, or just use some other maths library.

```javascript
    Visualiser.prototype.getAmplitude = function() {
        // Get time domain data
        var data = new Uint8Array(this.analyser.fftSize);
        this.analyser.getByteTimeDomainData(data);

        // Calculate peak
        const peak = maths.max(data);

        // Scale amplitude to between 0 and 1
        const amplitude = (peak - 128) / 128;
        return amplitude;
    };
```

### Smoothing it out
There is *one* problem with getting the amplitude like this and it's that, for most audio sources, it tends to change very quickly. When you're making a visualisation, rapid changes in a value will tend to make things jump and flicker and generally look nasty.

To fix this, let's interpolate between the values! What we'll do is store the amplitude when we measure it, and then next time we measure an amplitude, we'll do a bit of linear interpolation between this new amplitude and the last one. That way, there won't be as many sharp jumps in value.

We'll extract this bit of logic into its own little private function called `_smoothAmplitude`. Given an amplitude, and a linear interpolation factor (the distance the new value is from the last, as a fraction of the difference), the function returns a new, smoother amplitude.

```javascript
    Visualiser.prototype._smoothAmplitude = function(amp, factor = 0.15) {
        const smoothAmp = maths.lerp(this._previousAmplitude, amp, factor);
        this._previousAmplitude = smoothAmp;
        return smoothAmp;
    };
```

We can add a new method called `getAmplitudeSmooth` that makes use of this private smoothing function.

There you go; smoother than a fresh jar of Skippy™.

```javascript
    Visualiser.prototype.getAmplitudeSmooth = function(factor = 0.15) {
        const amp = this.getAmplitude();
        const smoothAmp = this._smoothAmplitude(amp, factor);
        return smoothAmp;
    };
```

And, if you really have to know - here's the linear interpolation function. Should be fairly intuitive. I think of it like: We want a point somewhere between `a` and `b`. The point should be some fraction, `t`, of the way between the two. So we start at `a` and add on a fraction `t` (so we need the product) of the distance between `a` and `b` (which is equal to `b` - `a`).

```javascript
    function lerp(a, b, t) {
        return a + (b - a) * t;
    }
```

# Making a simple visualisation
It's the moment you've all be waiting for... Let's visualise.

## The render function
Our visualiser allows us to pass in a custom 'render function' that actually does the drawing. Let's write a simple render function that, say, draws a square in the centre of the canvas that changes its size and colour according to the current amplitude.

The render function is automatically passed a reference to the `Visualiser`, and we can use this reference to get the rendering context. We can also call any of the methods we defined earlier on `Visualiser.prototype` to get the amplitude, or do whatever else.

We can make use of the same `lerp` function as before, this time for using the amplitude to control some other property such as colour hue in degrees or width and height in pixels.

This function will be called when each frame is to be drawn.

```javascript
    function ourRenderFunction(v) {
        // Get the rendering context
        const ctx = v.renderingContext;

        // Get the canvas width and height for easy access
        const canvasWidth = ctx.canvas.width;
        const canvasHeight = ctx.canvas.height;

        // Get amplitude
        const amp = v.getAmplitudeSmooth(0.15);

        // Calculate actual size of square (between 0 and 1000 pixels)
        const actualSize = maths.lerp(0, canvasHeight, amp);
        
        // Calculate position of origin of square
        const centreX = canvasWidth/2;
        const centreY = canvasHeight/2;

        const originX = centreX - actualSize/2;
        const originY = centreY - actualSize/2;

        // Clear canvas
        ctx.clearRect(0,0,canvasWidth,canvasHeight);

        // Set fill style
        ctx.fillStyle = `hsl(${maths.lerp(0, 360, amp)},100%,50%)`;

        ctx.fillRect(originX, originY, actualSize, actualSize);
    };
```

Earlier on, we defined our `<audio>` and `<canvas>` elements to have `id` values of 'in' and 'out', respectively. Now we can pass these `id`s to the `Visualiser` constructor to tie these two things together and make a visualiser!

Of course, we will also pass in our render function, sensibly named `ourRenderFunction`.

```javascript
    // Create a Visualiser instance that visualises the audio using the canvas
    const visualiser = new Visualiser('out', 'in', ourRenderFunction);
```

# The result
<iframe src="/experiments/webaudio-visualiser/rainbow.html" width="550" height="550" style="border:none;">Your browser doesn't support iframes. :(</iframe>

# Improvements

What we've set up so far works pretty nicely, but to make it even nicer, we could make some small additions.

These include:

- Non-linear interpolation (quadratic, cubic, quartic, polynomial)
- Retrieving frequency-domain data from the audio (using fourier analysis). WebAudio will actually help you with this. Just use `AnalyserNode.prototype.getByteFrequencyDomainData` or the equivalent float version.
- Some extra helper classes/functions for drawing shapes according to the current audio data.

# Here's one I made earlier
Anyway, enjoy. Here's another visualisation...

<iframe src="/experiments/webaudio-visualiser/web.html" width="550" height="550" style="border:none;">Your browser doesn't support iframes. :(</iframe>