# waveform-data.js [![Build Status](https://travis-ci.org/bbc/waveform-data.js.svg?branch=master)](https://travis-ci.org/bbc/waveform-data.js)

[![browser support](https://ci.testling.com/bbcrd/waveform-data.js.png)](https://ci.testling.com/bbcrd/waveform-data.js)

**waveform-data.js** is a JavaScript library for creating __zoomable__,
__browsable__ and __segmentable__ representations of audio waveforms.

**waveform-data.js** is part of a [BBC R&D Browser-based audio waveform visualisation software family](https://waveform.prototyping.bbc.co.uk):

- [audiowaveform](https://github.com/bbc/audiowaveform): C++ program that generates waveform data files from MP3 or WAV format audio.
- [audio_waveform-ruby](https://github.com/bbc/audio_waveform-ruby): A Ruby gem that can read and write waveform data files.
- **waveform-data.js**: JavaScript library that provides access to precomputed waveform data files, or can generate waveform data using the Web Audio API.
- [peaks.js](https://github.com/bbc/peaks.js): JavaScript UI component for interacting with waveforms.

We use these projects daily in applications such as
[BBC World Service Radio Archive](https://www.bbc.co.uk/rd/projects/worldservice-archive-proto) and __browser editing and sharing__ tools for BBC content editors.

![Example of what it helps to build](waveform-example.png)

# Install

## npm

You can use `npm` to install `waveform-data`, both for Node.js or your frontend needs:

```bash
npm install --save waveform-data
```

# Usage and Examples

Simply add `waveform-data.js` in a `script` tag in your HTML page.
Additional and detailed examples are showcased below and in the [documentation pages](doc/README.md).

```html
<!DOCTYPE html>
<html>
  <body>
    <script src="/path/to/waveform-data.js"></script>
    <script>
      var waveform = new WaveformData(...);
    </script>
  </body>
</html>
```

`dist/waveform-data.js` is delivered as UMD module so it can be used as:

- Vanilla JavaScript: available as `window.WaveformData`
- RequireJS module: available as `define(['WaveformData'], function(WaveformData) { ... })`
- CommonJS module: available as `const WaveformData = require('waveform-data');`

## Receiving data from an AJAX request

```javascript
const xhr = new XMLHttpRequest();

// .dat file generated by audiowaveform program
xhr.responseType = 'arraybuffer';
xhr.open("GET", 'https://example.com/waveforms/track.dat');

xhr.addEventListener('load', (progressEvent) => {
  const waveform = WaveformData.create(progressEvent.target);

  console.log(waveform.duration);
});

xhr.send();
```

## Receive data with `fetch`

[fetch](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch) is the next generation of data fetching for the web.

```javascript
fetch('http://example.com/waveforms/track.dat')
  .then(response => response.arrayBuffer())
  .then(buffer => {
    const waveform = WaveformData.create(buffer);

    console.log(waveform.duration);
  });
```

## Using Web Audio

Web Audio is an HTML5 API which can help fetch the waveform from the file directly. It is less optimised and more CPU intensive than consuming a waveform data file.

```javascript
const webAudioBuilder = require('waveform-data/webaudio');
const audioContext = new AudioContext();
```

### With AJAX

```javascript
xhr.open('GET', 'https://example.com/audio/track.ogg');
xhr.responseType = 'arraybuffer';

xhr.addEventListener('load', (progressEvent) => {
  webAudioBuilder(audioContext, progressEvent.target.response, (err, waveform) => {
    if (err) {
      console.error(err);
      return;
    }

    console.log(waveform.duration);
  });
});

xhr.send();
```

### With `fetch`

```javascript
fetch('https://example.com/audio/track.ogg')
  .then(response => response.arrayBuffer())
  .then(buffer => {
    webAudioBuilder(audioContext, buffer, (err, waveform) => {
      if (err) {
        console.error(err);
        return;
      }

      console.log(waveform.duration);
    });
  });
```

## Drawing in canvas

```javascript
const waveform = WaveformData.create(raw_data);

const interpolateHeight = (total_height) => {
  const amplitude = 256;
  return (size) => total_height - ((size + 128) * total_height) / amplitude;
};

const y = interpolateHeight(canvas.height);
const ctx = canvas.getContext('2d');
ctx.beginPath();

// from 0 to 100
waveform.min.forEach((val, x) => {
  ctx.lineTo(x + 0.5, y(val) + 0.5));
});

// then looping back from 100 to 0
waveform.max.reverse().forEach((val, x) => {
  ctx.lineTo((waveform.offset_length - x) + 0.5, y(val) + 0.5);
});

ctx.closePath();
ctx.stroke();
ctx.fill();
```

## Drawing in D3

```javascript
const waveform = WaveformData.create(raw_data);
const layout = d3.select(this).select('svg');
const x = d3.scale.linear();
const y = d3.scale.linear();
const offsetX = 100;

x.domain([0, waveform.adapter.length]).rangeRound([0, 1024]);
y.domain([d3.min(waveform.min), d3.max(waveform.max)]).rangeRound([offsetX, -offsetX]);

var area = d3.svg.area()
  .x((d, i) => x(i))
  .y0((d, i) => y(waveform.min[i]))
  .y1((d, i) => y(d));

graph.select('path')
  .datum(waveform.max)
  .attr('transform', () => `translate(0, ${offsetX})`)
  .attr('d', area);
```

## In Node.js

You can use the library to both consume the data on the frontend and emitting them from a Node.js HTTP server, for example.

```javascript
// app.js
const WaveformData = require('waveform-data');
const express = require('express');
const fs = require('fs');
const app = express();

// ...

app.get('/waveforms/:id.json', (req, res) => {
  res.set('Content-Type', 'application/json');

  fs.createReadStream(`path/to/${req.params.id}.json`)
    .pipe(res);
});
```

You could even self-consume the data from another application:

```javascript
#!/usr/bin/env node

// Save as: app/bin/cli-resampler.js

const WaveformData = require('waveform-data');
const request = require('superagent');
const args = require('yargs').argv;

request.get(`https://api.example.com/waveforms/${args.waveformid}.json`)
  .then(response => {
  const resampled_waveform = WaveformData.create(response.body).resample(2000);

  process.stdout.write(JSON.stringify({
    min: resampled_waveform.min,
    max: resampled_waveform.max
  }));
});
```

Usage: `./app/bin/cli-resampler.js --waveformid=1337`

# Data format

The file format used and consumed by `WaveformData` is documented [here](https://github.com/bbc/audiowaveform/blob/master/doc/DataFormat.md) as part of the [**audiowaveform** project](https://waveform.prototyping.bbc.co.uk).

# JavaScript API

This section describes the `WaveformData` API.

## `WaveformData`

This is the main object you use to interact with the waveform data. It helps you to:

* access the whole dataset
* iterate easily on an *offset* (visible subset of data, for example)
* generate one or several **resampled views**, e.g., to display the waveform at different zoom levels
* convert positions (in pixels, in seconds, in the offset)
* create and manage segments of waveform data, e.g., to represent different music tracks, or speakers, etc.

[`WaveformData` API Documentation](doc/WaveformData.md)

## `WaveformDataSegment`

Each segment of data is independent and can overlap other existing ones.
Segments allow you to keep track of portions of sound you would be interested to highlight.

[`WaveformDataSegment` API Documentation](doc/WaveformDataSegment.md)

## `WaveformDataAdapter`

This interface provides a backend abstraction for a `WaveformData` instance.
You should not manipulate this data directly.

[`WaveformDataArrayBufferAdapter` API Documentation](doc/WaveformDataArrayBufferAdapter.md)
[`WaveformDataObjectAdapter` API Documentation](doc/WaveformDataObjectAdapter.md)

# Browser Support

Any browser supporting **ECMAScript 5** will be enough to use the library -
think [`Array.forEach`](http://kangax.github.io/es5-compat-table/#Array.prototype.forEach):

 * IE9+, Firefox Stable, Chrome Stable, Safari 6+ are fully supported;
 * IE10+ is required for the [TypedArray](http://caniuse.com/#feat=typedarrays) Adapter;
 * Firefox 23+ and Webkit/Blink browsers are required for the experimental [Web Audio](http://caniuse.com/#feat=audio-api) Builder.

# Development

To develop the code, install [Node.js](http://nodejs.org/) and [npm](https://npmjs.org/). After obtaining the waveform-data.js source code, run `npm install` to install Node.js package dependencies.

# Credits

This program contains code adapted from [Audacity](http://audacity.sourceforge.net/), used with permission.

# License

See COPYING for details.

# Contributing

Every contribution is welcomed, either it's code, idea or a *merci*!

[Guidelines are provided](CONTRIBUTING.md) and every commit is tested against unit tests
using [Karma runner](http://karma-runner.github.io) and the [Chai assertion library](http://chaijs.com/).

# Copyright

Copyright 2018 British Broadcasting Corporation
