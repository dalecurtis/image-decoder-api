# ImageDecoder Explainer

## Authors:

- Dale Curtis @ Google

## Participate:

- [Issue Tracker](https://github.com/dalecurtis/image-decoder-api/issues)
- [Prototype](https://chromium-review.googlesource.com/c/chromium/src/+/2145133)
- TODO: Link to discourse here.

## Introduction
Today [`<img>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement) elements don't provide access to any frames beyond the first. They also provide no control over which frame is displayed in an animation. As we look to provide audio and video codecs through [WebCodecs](https://github.com/WICG/web-codecs/blob/master/explainer.md) we should consider similar interfaces for images as well.

We propose a new ImageDecoder API to provide web authors access to an [ImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap) of each frame given an arbitrary byte array as input. The returned ImageBitmaps can be used for drawing to canvas or WebGL (as well as any other future ImageBitmap use cases). Since the API is not bound to the DOM it may also be used in workers.

## Goals
* Providing explicit control over decoded images and their display.
* Extracting a given frame (or sequence of frames) from animated images.
* Usage of image decoding in out of DOM scenarios (offscreen worker, etc).

## Non-Goals
* Defining how authors may provide their own decoders for formats that are unsupported by the user agent.
  * E.g., &lt;img src="cats.pcx"&gt;.
* Defining an ImageEncoder API; that's left for another explainer.

## ImageDecoder API

### Example 1: Animated GIF Renderer

```Javascript
// This example renders an animated image to a canvas via ReadableStream.

let canvas = document.createElement('canvas');
let canvasContext = canvas.getContext('2d');
let imageDecoder = null;
let imageIndex = 0;

function renderImage(imageFrame) {
  canvasContext.drawImage(imageFrame.image, 0, 0);
  if (imageDecoder.frameCount == 1)
    return;

  if (imageIndex + 1 >= imageDecoder.frameCount)
    imageIndex = 0;

  // Decode the next frame ahead of display so it's ready in time.
  imageDecoder.decode(++imageIndex).then(nextImageFrame => setTimeout(
      _ => { renderImage(nextImageFrame); }, imageFrame.duration / 1000.0));
}

function decodeImage(imageByteStream) {
  imageDecoder = new ImageDecoder({data: imageByteStream, options: {}});
  console.log('imageDecoder.frameCount = ' + imageDecoder.frameCount);
  console.log('imageDecoder.type = ' + imageDecoder.type);
  console.log('imageDecoder.repetitionCount = ' + imageDecoder.repetitionCount);
  imageDecoder.decode(imageIndex).then(renderImage);
}

fetch("animated.gif").then(response => decodeImage(response.body));
```

Output:
```Text
imageDecoder.frameCount = 20
imageDecoder.type = "image/gif"
imageDecoder.repetitionCount = 0
```
![Example](test-gif.gif)


### Example 2: Inverted MJPEG Renderer
```Javascript
// This example renders a multipart/x-mixed-replace MJPEG stream to canvas.

let canvas = document.createElement('canvas');
let canvasContext = canvas.getContext('2d');

function decodeImage(imageArrayBufferChunk) {
  // JPEG decoders don't have the concept of multiple frames, so we need a new
  // ImageDecoder instance for each frame.
  let imageDecoder = new ImageDecoder(
      {data: imageArrayBufferChunk, options: {imageOrientation: "flipY"}});
  console.log('imageDecoder.frameCount = ' + imageDecoder.frameCount);
  console.log('imageDecoder.type = ' + imageDecoder.type);
  console.log('imageDecoder.repetitionCount = ' + imageDecoder.repetitionCount);
  imageDecoder.decode(imageIndex).then(
      imageFrame => canvasContext.drawImage(imageFrame.image, 0, 0));
}

fetch("https://mjpeg_server/mjpeg_stream").then(response => {
  const contentType = response.headers.get("Content-Type");
  if (!contentType.startsWith("multipart"))
    return;

  let boundary = contentType.split("=").pop();

  // See https://github.com/whatwg/fetch/issues/1021#issuecomment-614920327
  let parser = new MultipartParser(boundary);
  parser.onChunk = arrayBufferChunk => decodeImage(arrayBufferChunk);

  let reader = response.body.getReader();
  reader.read().then(function getNextImageChunk({done, value}) {
    if (done) return;
    parser.addBinaryData(value);
    return reader.read().then(getNextImageChunk);
  });
});
```

Output:
```Text
imageDecoder.frameCount = 1
imageDecoder.type = "image/jpeg"
imageDecoder.repetitionCount = 0
...
```
![Example](flipped-gif.gif)

## Open Questions / Notes / Links
* image/svg support is not currently possibly in Chrome since it's bound to DOM.
* Using a ReadableStream may over time accumulate enough data to cause OOM.
* Should we allow mime sniffing at all? It's [discouraged](https://github.com/dalecurtis/image-decoder-api/issues/1) these days, but &lt;img&gt; has historically depended on it.
* Is there more EXIF information that we'd want to expose?

## Considered alternatives

### Providing image decoders through the VideoDecoder API.
The VideoDecoder API being designed for WebCodecs is intended for transforming demuxed encoded data chunks into decoded frames. Which is problematic for image formats since generally their containers and encoded data are tightly coupled. E.g., you don't generally have a gif demuxer and a gif decoder, just a decoder.

If we allow VideoDecoder users to enqueue raw image blobs we'll have to output all contained frames at once. Without external knowledge of frame locations within the blob, users will have to decode batches of unknown size or decode everything at once. I.e., there is no piece-wise decoding of an arbitrarily long image sequence and users need to cache all decoded outputs. This feels bad from a utility and resource usage perspective.

The current API allows users to provide as much or as little data as they want. Images are not decoded until needed. Users don't need to cache their decoded output since they have random access to arbitrary images.

Other minor cumbersome details:
* Image containers may define image specific fields like repetition count.
* Image containers typically have complicated ICC profiles which need application.

### Hang the API off Image/Picture elements
This is precluded due to our goal of having the API work out of DOM.

## Stakeholder Feedback / Opposition

- Chrome : Positive
- Developers : Positive
- Other user agents : No signals

## Proposed IDL

```Javascript
dictionary ImageFrame {
  // Actual decoded image; includes resolution information.
  required ImageBitmap image;

  // Expected on screen duration for the image in microseconds.
  required unsigned long long duration;

  // JEITA CP-3451 EXIF orientation code.
  required unsigned long orientation;

  // TODO: Color space information?
};

typedef (BufferSource or ReadableStream) ImageBufferSource;
dictionary ImageDecoderInit {
  ImageBufferSource data;

  // See https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#imagebitmapoptions
  ImageBitmapOptions options;
};

interface ImageDecoder {
  constructor(ImageDecoderInit init);

  // Decodes the frame at the given index. If we're still receiving data, this
  // method will wait to resolve the promise until the given |frameIndex| is
  // available or reject the promise if we receive all data or fail before
  // |frameIndex| is available.
  Promise<ImageFrame> decode(unsigned long frameIndex);

  // The number of frames in the image.
  readonly attribute unsigned long frameCount;

  // The detected mime type for the decoded image.
  readonly attribute USVString type;

  // The image's preferred repetition count.
  readonly attribute unsigned long repetitionCount;
};
```
