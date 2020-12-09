---
layout: post
title: "Use AudioContext and do some pitch shifting"
---

# [](#AudioContext-javascript-pitch-shifting) JavaScript, AudioContext, Pitch Shifting

After the last adventure into using audio with JavaScript I wanted to learn more about manipulating the microphone with the browser.  I gave myself a goal to see if it is possible to pitch shift the audio live!


### [](#step-one-get-audio)Step One Get Audio Stream

The first step to pitch shift is to actually use JavaScript to stream the user's mic in the browser.

> Requirements
>
> Using JavaScript grab the microphone stream and let the browser play it on the page.
>
> Note: Most browsers require that the user preform an action before you can access the mic.

```html
<button id="get-mic">Get Mic!</button>
```

```js
let mediaStreamSource;
let audioContext;

// create the audio context then connect the stream to our audio destination (browser)
function processAudioStream(stream) {
    audioContext = new AudioContext();
    mediaStreamSource = audioContext.createMediaStreamSource(stream);
    mediaStreamSource.connect(audioContext.destination);
}

// Grab the media devices and passing in audio since we just want the mic
// We pass the audio stream to our process function
document.querySelector("#get-mic").addEventListener(
  "click",
  () => {
    navigator.mediaDevices
      .getUserMedia({ audio: true })
      .then(processAudioStream);
  },
  { once: true }
);

```

Great now we have the mic playing in our browser so let's pitch shift it!

### [](#step-two-pitch-shift)Step Two Pitch Shift

Sadly pitch shifting ended up being a bit more advanced than I thought but I found some code off of github that does the effect.  I included the link to the github in my example for reference.

Another thing we can implement is a slider to change pitch effect at our leasure.

```html
<button id="shift-it">Pitch Shift!</button>
<div>
	<input type="range" id="pitch" name="pitch" min="-2" max="2" value="-.6" step="0.1">
	<label for="pitch">Pitch</label>
</div>
```

```js
let mediaStreamSource;
let pitchChangeEffect;
window.AudioContext = window.AudioContext || window.webkitAudioContext;
let audioContext;

function processAudioStream(stream) {
  audioContext = new AudioContext();
  mediaStreamSource = audioContext.createMediaStreamSource(stream);
  // https://github.com/cwilso/Audio-Input-Effects/blob/master/js/jungle.js
  pitchChangeEffect = new Jungle(audioContext);
  mediaStreamSource.connect(pitchChangeEffect.input);
  pitchChangeEffect.output.connect(audioContext.destination);
}

document.querySelector("#shift-it").addEventListener(
  "click",
  () => {
    navigator.mediaDevices
      .getUserMedia({ audio: true })
      .then(processAudioStream);
  },
  { once: true }
);

// Using the input event from our html5 slider we can change the pitch of our mic!
document.querySelector("#pitch").addEventListener("input", e => {
  if (!pitchChangeEffect) return;
  let offset = e.target.value;
  pitchChangeEffect.setPitchOffset(offset);
});
```
This was actually really fun to make and I spent way too long playing with it.

See this code in action and have fun with the pitch changing [stackblitz](https://stackblitz.com/edit/pitch-shift)

Now there are lots of things you can do with the audioContext api but don't take my work for it find out for yourself! 
[Github with more examples](https://github.com/Robooto/js-pitch-shift)
[AudioContext](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext)