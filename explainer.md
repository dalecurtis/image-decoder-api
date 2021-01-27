# ImageDecoder Explainer

## Authors:

- Dale Curtis @ Google

## Participate:

- [Issue Tracker](https://github.com/dalecurtis/image-decoder-api/issues)
- [Prototype](https://chromium-review.googlesource.com/c/chromium/src/+/2145133)
- [Discourse](https://discourse.wicg.io/t/proposal-imagedecoder-api-extension-for-webcodecs/4418)

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
* Defining an ImageEncoder API; that's left for another explainer and is already provided somewhat by the [Canvas.toBlob() API](https://developer.mozilla.org/en-US/docs/Web/API/HTMLCanvasElement/toBlob).

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
  imageDecoder.decode({frameIndex: ++imageIndex}).then(
      nextImageFrame => setTimeout(_ => {
          renderImage(nextImageFrame); }, imageFrame.duration / 1000.0));
}

function logMetadata(imageDecoder) {
  if (imageDecoder.frameCount == 1) {
    console.log('Still image:');
  else
    console.log('Animated image:');
  console.log('imageDecoder.type = ' + imageDecoder.type);
  console.log('imageDecoder.complete = ' + imageDecoder.complete);
  console.log('imageDecoder.frameCount = ' + imageDecoder.frameCount);
  console.log('imageDecoder.repetitionCount = ' + imageDecoder.repetitionCount);
  console.log('imageDecoder.tracks.length = ' + imageDecoder.tracks.length);
  for (var i = 0; i < imageDecoder.tracks.length; ++i) {
    console.log(`imageDecoder.tracks[${i}] = ` +
        JSON.stringify(imageDecoder.tracks[i]));
  }
}

function decodeImage(imageByteStream) {
  imageDecoder = new ImageDecoder(
      {data: imageByteStream, type: "image/gif", options: {}});
  logMetadata(imageDecoder);
  imageDecoder.decode({frameIndex: imageIndex}).then(renderImage);
}

fetch("animated.gif").then(response => decodeImage(response.body));
```

Output:
```Text
Animated image:
imageDecoder.type = "image/gif"
imageDecoder.complete = true
imageDecoder.frameCount = 20
imageDecoder.repetitionCount = 0
imageDecoder.tracks.length = 1;
imageDecoder.tracks[0] = {"animated":true,"id":0}
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
  let imageDecoder = new ImageDecoder({
    data : imageArrayBufferChunk,
    type : "image/jpeg",
    options : {imageOrientation : "flipY"}
  });
  logMetadata(imageDecoder);
  imageDecoder.decode({frameIndex: imageIndex}).then(
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
Still image:
imageDecoder.type = "image/jpeg"
imageDecoder.complete = true
imageDecoder.frameCount = 1
imageDecoder.repetitionCount = 0
imageDecoder.tracks.length = 1;
imageDecoder.tracks[0] = {"animated":false,"id":0}
...
```
![Example](flipped-gif.gif)


### Example 3: Multi-track image selection.
```Javascript
// This example renders the animation track of a multi-track image after
// initially selecting the still image.

let canvas = document.createElement('canvas');
let canvasContext = canvas.getContext('2d');
let imageDecoder = null;
let imageIndex = 0;

function decodeImage(imageByteStream) {
  // preferAnimation=false ensures we select the still image instead of whatever
  // the container metadata might want us to select instead.
  imageDecoder = new ImageDecoder(
      {data: imageByteStream, type: "image/avif", preferAnimation: false});

  // This step isn't necessary, but shows how you can get metadata before any
  // frames have been decoded.
  imageDecoder.decodeMetadata().then(logMetadata);

  // Start decoding of the first still image.
  imageDecoder.decode({frameIndex: imageIndex}).then(image => {
    renderImage(image);
    if (imageDecoder.frameCount > 1)
      return;

    // Identify the first animated track.
    var animationTrackId = -1;
    for (var i = 0; i < imageDecoder.tracks.length; ++i) {
      if (imageDecoder.tracks[i].animated) {
        animationTrackId = i;
        break;
      }
    }

    if (animationTrackId == -1)
      return;

    // Switch to the animation track.
    imageDecoder.selectTrack(animationTrackId);
    logMetadata(imageDecoder);

    // Start decoding loop for the animation track.
    imageIndex = 0;
    imageDecoder.decode({frameIndex: imageIndex}).then(renderImage);
  });
}

fetch("animated_and_still.avif").then(response => decodeImage(response.body));
```

Output:
```Text
Still image:
imageDecoder.type = "image/avif"
imageDecoder.complete = true
imageDecoder.frameCount = 1
imageDecoder.repetitionCount = 0
imageDecoder.tracks.length = 2;
imageDecoder.tracks[0] = {"animated":false,"id":0}
imageDecoder.tracks[1] = {"animated":true,"id":1}

Animated image:
imageDecoder.type = "image/avif"
imageDecoder.complete = true
imageDecoder.frameCount = 20
imageDecoder.repetitionCount = 0
imageDecoder.tracks.length = 2;
imageDecoder.tracks[0] = {"animated":false,"id":0}
imageDecoder.tracks[1] = {"animated":true,"id":1}
...
```
![Example](test-still.png) ![Example](test-gif.gif)

## Open Questions / Notes / Links
* image/svg support is not currently possibly in Chrome since it's bound to DOM.
* Using a ReadableStream may over time accumulate enough data to cause OOM.
* Is there more EXIF information that we'd want to expose?
* Should we take a fetch() [Request](https://developer.mozilla.org/en-US/docs/Web/API/Request) object as input so that the API can allow display with tainting of image data that would be blocked by CORS?

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
- Firefox : Positive (unofficially)

## Proposed IDL

```Javascript
dictionary ImageFrame {
  // Actual decoded image; includes resolution information.
  required ImageBitmap image;

  // Indicates if the decoded image is actually complete.
  required boolean complete;

  // Expected on screen duration for the image in microseconds.
  required unsigned long long duration;

  // JEITA CP-3451 EXIF orientation code.
  required unsigned long orientation;

  // TODO: Color space information?
};

typedef (ArrayBuffer or ArrayBufferView or ReadableStream) ImageBufferSource;
dictionary ImageDecoderInit {
  required ImageBufferSource data;

  // Mime type for |data|. Providing the wrong mime type will lead to a decoding
  // failure.
  required USVString type;

  // Options to use when creating ImageBitmap objects from decoded frames. The
  // resize width and height are additionally used to facilitate reduced
  // resolution decoding.
  //
  // See https://html.spec.whatwg.org/multipage/imagebitmap-and-animations.html#imagebitmapoptions
  ImageBitmapOptions options;

  // For multi-track images, indicates that the animation is preferred over any
  // still images that are present. When unspecified the decoder will use hints
  // from the data stream to make a decision.
  boolean preferAnimation;
};

dictionary ImageDecodeOptions {
  // The index of the frame to decode.
  unsigned long frameIndex = 0;

  // When |completeFramesOnly| is set to false, partial progressive frames will
  // be returned. When in this mode, decode() calls will resolve only once per
  // new partial image at |frameIndex| until the frame is complete.
  boolean completeFramesOnly = true;
};

interface ImageDecoder {
  [RaisesException] constructor(ImageDecoderInit init);

  // Returns true if ImageDecoder supports decoding of the given mime type.
  static boolean canDecodeType(USVString type);

  // Decodes a frame using the given |options| or the first frame if no options
  // are provided. If data is still being received, the promise won't be
  // resolved or rejected until the given |options.frameIndex| is available,
  // all data is received, or a decoding error occurs.
  Promise<ImageFrame> decode(optional ImageDecodeOptions options);

  // Decodes only the metadata for an image; resolves the promise when metadata
  // can be decoded. Normally this is done automatically at construction time.
  // However when using a ReadableStream, there may not be enough data to decode
  // metadata at the time of construction.
  //
  // Unfortunately GIFs don't really have metadata, so ReadableStream use cases
  // may have inaccurate |frameCount| and |tracks.animated| status even after
  // the returned promise is resolved. |frameCount| won't be accurate until
  // |complete| is true. |tracks.animated| won't be accurate until |complete| is
  // true or more than one frame is received.
  Promise<void> decodeMetadata();

  // Selects another track of the image. Destructively recreates the underlying
  // decoder. Identical track selections will be ignored. Invalid track
  // selections will raise an exception. Clears the |preferAnimation| flag if
  // one was provided during construction.
  //
  // Changing tracks will resolve all outstanding decode requests as rejected
  // and reset any partially decoded frame state. Outstanding ImageFrames and
  // metadata decode promises will remain valid.
  [RaisesException] void selectTrack(unsigned long trackId);

  // The number of frames in the image.
  //
  // When decoding a ReadableStream the value will be 0 until enough data to
  // decode metadata has been received. If the format has no fixed count, the
  // value will increase as frames are received by the decoder.
  readonly attribute unsigned long frameCount;

  // The mime type for the decoded image. This reflects the value provided
  // during construction.
  readonly attribute USVString type;

  // The image's preferred repetition count.
  //
  // When decoding a ReadableStream the value will be 0 until enough data to
  // decode metadata has been received.
  readonly attribute unsigned long repetitionCount;

  // True if all available frames have been received by the decoder.
  readonly attribute boolean complete;

  // List of tracks available in this image.
  //
  // When decoding a ReadableStream the array will be empty until enough data to
  // decode metadata has been received.
  readonly attribute FrozenArray<ImageTrack> tracks;
};
```
