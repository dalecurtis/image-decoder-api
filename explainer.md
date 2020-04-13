# ImageDecoder Explainer

# Introduction
Today [`<img>`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLImageElement) elements don't have a way to provide access to frames beyond the first. They also have no control over which frame is displayed in an animation.

As we look to provide audio and video decoders through [WebCodecs](https://github.com/WICG/web-codecs/blob/master/explainer.md)it makes sense to consider similar interfaces for images as well.

We propose a new ImageDecoder API to provide WebAuthors access to an [ImageBitmap](https://developer.mozilla.org/en-US/docs/Web/API/ImageBitmap) of each frame given a arbitrary byte array.

If we choose to pursue this API, it likely makes sense to provide an ImageEncoder interface as well, but that topic is left for a later discussion.


# Use cases
* Explicit control over decoded images and their display.
* Extracting a given frame (or sequence of frames) from an animated image.


# Proposed API

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

dictionary ImageDecoderInit {
  BufferSource imageData;

  // Optional parameter indicating that color space information for the decoded
  // image should be ignored.
  boolean? ignoreColorSpace;

  // Optional desired resolution parameters which can be used by some decoders
  // to implement reduced resolution decoding. Not all decoders support this;
  // when unsupported, images will be returned at full resolution.
  unsigned long? desiredWidth;
  unsigned long? desiredHeight;
};

interface ImageDecoder {
  [CallWith=ScriptState, RaisesException] constructor(ImageDecoderInit init);

  // Decodes the frame at the given index.
  Promise<ImageFrame> decode(unsigned long frameIndex);

  // The number of frames in the image.
  readonly attribute unsigned long frameCount;

  // The detected mime type for the decoded image.
  readonly attribute DOMString mimeType;

  // The image's preferred repetition count.
  readonly attribute unsigned long repetitionCount;
};
```

# Example

```Javascript
  // This example renders an animated image to a canvas.

  let canvas = document.createElement('canvas');
  let canvasContext = canvas.getContext('2d');

  let data = /* ArrayBuffer or ArrayBufferView of image data bytes */;
  let imageDecoder = new ImageDecoder({imageData: data});
  console.log('imageDecoder.frameCount = ' + imageDecoder.frameCount);
  console.log('imageDecoder.mimeType = ' + imageDecoder.mimeType);

  let imageIndex = 0;
  function renderImage(imageFrame) {
    canvasContext.drawImage(imageFrame.image, 0, 0);
    if (imageIndex + 1 >= imageDecoder.frameCount)
      imageIndex = 0;

    // Decode the next frame ahead of display so it's ready in time.
    imageDecoder.decode(++imageIndex).then(nextImageFrame => setTimeout(
        _ => { renderImage(nextImageFrame); }, imageFrame.duration / 1000.0));
  }

  imageDecoder.decode(imageIndex).then(renderImage);
```

Output:
```Text
  imageDecoder.frameCount = 20
  imageDecoder.mimeType = "image/gif"
```
![Example](test-gif.gif)


# Open Questions / Notes / Links
* Should this be a new API or just hang off of the ImageElement?
* Is there more EXIF information that we'd want to expose?
* Should color correction and format conversion happen automatically? Optionally? [ImageBitmapOptions](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/createImageBitmap#Syntax) offers one solution.
* [Link to GitHub repository.](https://github.com/dalecurtis/image-decoder-api/blob/master/explainer.md)
* [Link to Chromium Prototype.](https://chromium-review.googlesource.com/c/chromium/src/+/2145133)
