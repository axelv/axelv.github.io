---
title: "A simple pitch detector in React"
author: "Axel Vanraes"
categories:
  - "react"
  - "pitch detection"
  - "audio"
  - "music"
---

Demo: [https://whatpitchisthis.vercel.app/](https://whatpitchisthis.vercel.app/)

In between coding for healthcare, I like to unwind by playing jazz on the piano. This time I found and even more exciting project combining engineering and music: building a simple pitch detector in React. This is also an excellent opportunity to explore the intricacies of audio processing in the browser.

# Setting up the project

To initiate the project, I opted for Vite and made sure to include the basicSsl plugin for enabling microphone access in the browser. Chrome mandates a secure connection for microphone access.

```bash
npx create-vite-app chord-detector --template react-ts
```

```bash

yarn add --dev @vitejs/plugin-basic-ssl
```

```js
// vite.config.ts
export default defineConfig({
  plugins: [react(), basicSsl()],
});
```

## Creating the `usePitchDetector` hook

In this post, I'll skip over styling and UI, focusing instead on (1) the logic controlling pitch detection and (2) the pitch detection algorithm.

The `usePitchDetector` hook should return four things:

1. `start()`: a function to start the pitch detection
2. `stop()`: a function to stop the pitch detection
3. `hasStarted`: a boolean indicating wether the pitch detection has started
4. `note`: a string indicating the detected note

Setting up the audio context

To retrieve raw audio samples from the microphone, we need two crucial objects: an `AudioContext` and an `AnalyserNode`. The `AudioContext` is the primary object managing audio processing, while the `AnalyserNode` allows us to fetch audio samples in both the time and frequency domain.

Before creating the `AudioContext`, we need to confirm browser support for the `getUserMedia` API, which grants microphone access. If the browser lacks support, an error message is logged, and the function returns.

The following code illustrates the setup:

```tsx
const [analyser, setAnalyser] = useState<AnalyserNode | null>(null);
/**
 * Start the recording
 * Create the analyser and connect it to the microphone. This will start the pitch detection.
 */
const start = () => {
  if (navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
    const audioContext = new AudioContext();
    console.log("getUserMedia supported.");
    navigator.mediaDevices
      .getUserMedia(
        // constraints - only audio needed for this app
        {
          audio: {
            // @ts-expect-error - typescript doesn't know about the goog properties
            mandatory: {
              googEchoCancellation: "false",
              googAutoGainControl: "false",
              googNoiseSuppression: "false",
              googHighpassFilter: "false",
            },
            optional: [],
          },
        }
      )
      // Success callback
      .then((stream) => {
        const mediaStreamSource = audioContext.createMediaStreamSource(stream);
        // Connect it to the destination.
        const analyser = audioContext.createAnalyser();
        analyser.fftSize = 2048;
        mediaStreamSource.connect(analyser);
        setAnalyser(analyser);
      })
      // Error callback
      .catch((err) => {
        console.error(`The following getUserMedia error occurred: ${err}`);
      });
  } else {
    console.log("getUserMedia not supported on your browser!");
  }
};
```

Remarks:

The outer if-statement checks for `getUserMedia`` API support; otherwise, an error message is logged.

The `getUserMedia` API returns a `Promise` resolving to a `MediaStream` object, a stream of audio samples. This object is then used to create a `MediaStreamSource`, allowing access to audio samples from the microphone.

The `AudioContext` connects the `MediaStreamSource`, initiating pitch detection.

The `AnalyserNode` is created and connected to the `MediaStreamSource`. The `fftSize` property determines the number of samples retrieved, balancing accuracy and performance.

The `AnalyserNode` is stored in the state for later access.

1. Finally we save the `AnalyserNode` in the state so we can access it later.

### Continously processing audio samples and calculating the pitch

Once the analyser state is set, we can begin processing audio samples. The `requestAnimationFrame` API facilitates continuous processing, as demonstrated below:

```tsx
// Keep track of the requestAnimationFrame id
const requestRef = React.useRef<number | null>(null);

/**
 * Update the pitch
 */
const updatePitch = (
  analyser: AnalyserNode,
  options: { sampleRate: number }
) => {
  const buf = new Float32Array(analyser.fftSize);
  analyser.getFloatTimeDomainData(buf);
  const pitch = calculatePitch(buf, { sampleRate });
  // some update logic
  requestRef.current = window.requestAnimationFrame(() =>
    updatePitch(analyser, options)
  );
};

/**
 * Continously process the audio samples and calculate the pitch
 */
useEffect(() => {
  if (analyser) {
    const options = {
      sampleRate: analyser.context.sampleRate,
    };
    updatePitch(analyser, options);
    return () => {
      requestRef.current && window.cancelAnimationFrame(requestRef.current);
    };
  }
}, [analyser]);
```

**Remarks:**

1. The `useRef` hook tracks the last `requestAnimationFrame` id, allowing cancellation when the component unmounts.

2. The `updatePitch` function is scheduled using the `requestAnimationFrame` API, creating a loop for continuous audio sample processing.

3. The `updatePitch` function populates a buffer with audio samples in the time domain using `getFloatTimeDomainData`. The buffer is t`en passed to the `calculatePitch` function for pitch calculation.

4. The `useEffect` hook initiates pitch detection when the `analyser` state is set and cancels the animation frame on component unmount

### Stop the pitch detection

To conclude pitch detection, we call the `cancelAnimationFrame` API and set the analyser state to `null`:

```tsx
/**
 * Stop the recording
 * Cancel the animation frame and set the analyser to null
 */
const stop = () => {
  requestRef.current && window.cancelAnimationFrame(requestRef.current);
  setAnalyser(null);
};
```

## The Pitch Detection Algorithm

Now equipped with the `usePitchDetector` hook, we can delve into the pitch detection algorithm. This algorithm assumes the pitch equals the fundamental frequency, a reasonable approximation for most instruments.

### Auto-correlation

At the heart of finding the fundamental frequency lies the task of discerning the periodicity within the audio signal—simply put, understanding how much time elapses before the signal repeats itself.

The auto-correlation function acts as our guide, measuring the resemblance between the original signal and its shifted versions. This process becomes instrumental in isolating the periodic patterns within the signal:

```ts
/*``
 * Calculate the auto-correlation of the signal
 */
function acf(values: number[] | Float32Array) {
  const SIZE = values.length;
  const c = new Array(SIZE).fill(0);
  // iterate all lags
  for (let i = 0; i < SIZE; i++)
    // iterate overlapping windows
    for (let j = 0; j < SIZE - i; j++) c[i] = c[i] + values[j] * values[j + i];
  return c;
}
```

### Finding the maximum score

The maximum score in the auto-correlation function reveals the signal's periodicity. The following function identifies the maximum score:

```ts
/**
 * Find the maximum score in the auto-correlation function
 */
function findMax(c: number[]) {
  let max = 0;
  let maxIndex = -1;
  for (let i = 0; i < c.length; i++) {
    if (c[i] > max) {
      max = c[i];
      maxIndex = i;
    }
  }
  return { max, maxIndex };
```

**Remarks:**
To enhance precision, the sample index of the maximum score is approximated using parabolic interpolation:

```ts
/**
 * Approximate the peak with a parabola and find the maximum of the parabola
 */
function parabolicInterpolation(c: number[], i: number) {
  const y0 = c[i - 1];
  const y1 = c[i];
  const y2 = c[i + 1];
  const a = (y0 + y2) / 2 - y1;
  const b = (y2 - y0) / 2;
  if (a === 0) {
    return i;
  } else {
    return i - b / (2 * a);
  }
}
```

## Putting it all together

With the `usePitchDetector` hook and the pitch detection algorithm in place, we can integrate them into a simple React app.

You can find the full code on [Github](https://github.com/axelv/whatpitchisthis). A demo is available [here](https://whatpitchisthis.vercel.app/).
