# Extra Timestamps for encoded RTC media frames

## Authors:

- Palak Agarwal (Google)
- Guido Urdaneta (Google)

## Participate
- https://github.com/w3c/mediacapture-extensions


## Introduction

The [Screen Capture](https://w3c.github.io/mediacapture-screen-share/)
API allows users to share their screen, typically in videoconferencing
scenarios that involve [WebRTC](https://w3c.github.io/webrtc-pc/) to send
the screen capture together with other audio and video to remote users.
The screen capture may consist of the full screen, but also an application
window or a browser tab. The captured content (screen, window or tab) is
often referred to as a captured surface.

The video from a screen capture is returned as a MediaStreamTrack, which
provides an API surface to control the media flow and to retrieve properties
about the captured surface. These properties are reported as a MediaTrackSettings
object, returned by the getSettings method.

This feature consists in adding a number of additional properties to the
MediaTrackSettings object returned by getSettings() containing the physical
and logical resolutions of the video. An event is also fired when these
resolutions change during the capture session.

Currently, MediaTrackSettings contains the actual resolution of the frames
flowing through the MediaStreamTrack via the width and height properties.
The physical resolution of a captured surface is the actual size of the surface
in pixels. Very often, the physical resolution is the same as the already
reported resolution (width/height properties). However, an application can
alter this by rescaling the video using constraints.

In some cases, a surface's resolution is altered by zooming provided by the
operating system and/or the Web browser (in the case of tabs). The result can
be that a surface with a relatively small size can have a high resolution due
to the effect of zooming. In this case, a videoconferencing application can 
optimize bandwidth consumption by sending the video over the network using
a lower resolution. Knowing the physical and logical resolution of the captured
surface helps the application make this determination and apply this
optimization.

## User-Facing Problem

The basic problem to solve is optimizing the bandwidth used to transmit the
video from a screen-capture session over a WebRTC peer connection in cases
when the captured surface may be heavily zoomed in. In this cases, the
resolution of the surface is high (which would be costly), but the actual
content can be faithfully transmitted at a lower resolution, which requires
fewer resources.

### Goals

- Provide Web applications using Screen Capture access to the physical and
  logical resolution of a captured surface.

### Non-goals

- Provide mechanisms to improve WebRTC communication mechanisms based on the
information provided by this feature.


### Example

This shows an example of an application that:
1. Computes the delay between audio and video
2. Computes the processing and logs and/or updates remote parameters based on the
delay.

```js


const stream = await navigator.mediaDevices.getDisplayMedia({video: true});
const track = stream.getVideoTracks()[0];
const settings = track.getSettings();
let physicalSize = settings.physicalWidth * settings.physicalHeight;
let logicalSize = settings.logicalWidth * settings.logicalHeight;
maybeAdjustResolution(track);

track.onconfigurationchange = () => {
  const oldPhysicalSize = physicalSize;
  const oldLogicalSize = logicalSizel
  const settings = track.getSettings();
  physicalSize = settings.physicalWidth * settings.physicalHeight;
  logicalSize = settings.logicalWidth * settings.logicalHeight;
  maybeAdjustResolution(track)
}

function maybeAdjustResolution(track) {
  if (physicalSize / logicalSize > Constant) { // Constant is app-defined
    await track.applyConstraints({width: settings.logicalWidth, height: settings.logicalHeight});
  }
}

## Alternatives considered

### [Alternative 1]

Expose the values in CaptureController. CaptureController is an object uniquely
associated with a screen-capture session and, since the physical and logical
resolutions are properties of the captured surface, they would be a good fit
for CaptureController. However, exposing properties associated with the source
of a MediaStreamTrack via MediaTrackSettings is a well established pattern.
Moreover, the related width and height properties are also exposed as
MediaTrackSettings.


### [Alternative 2]

Rely on existing APIs that provide a device pixel ratio, such as 
[window.devicePixelRatio](https://drafts.csswg.org/cssom-view-1/#dom-window-devicepixelratio)
or the [Window Management API](https://w3c.github.io/window-management/#screen-device-pixel-ratio). 
The device pixel ratio is the ratio between the size of a CSS pixel and the size
of a physical pixel.

These APIs can help solve the problem in some situations, but not all.
window.devicePixelRatio is limited to the surface where the script runs, which
is not necessarily the captured surface. 
[ScreenDetailed.devicePixelRatio](https://w3c.github.io/window-management/#screen-device-pixel-ratio)
would be useful for a number of single-screen cases, but in a multiple-screen
environment it would not be possible to know which screen corresponds to the
captured surface in a multiple-screen environment.

### [Alternative 3]
Do image processing to determine the level of zoom in of the captured surface.
Since the application has access to all the pixels of the captured surface, it
should be possible to apply image-processing techniques on each frame to
determine if it is suitable to be downscaled without losing quality for network
transmission. 

The problem with this approach is that it is costly in terms of resources and
less accurate than accessing the exact ratio.

## Accessibility, Privacy, and Security Considerations

This API makes it possible to derive the zoom level (largely equivalent to the
device pixel ratio) of a captured surface by comparing its logical resolution
and its physical resolution.
This adds a small amount of fingerprinting entropy, but only in the case when
the user has explicitly authorized the capture of the surface, which provides
access to all the pixels and thus the possibility of determining this value
via image processing.
We consider that this amount of extra information is the minimum required to
support the intended uses case and the risk is sufficiently mitigated by
tying the exposure to the display-capture permission, which requires an explicit
dialog, is not persitent, is limited to the captured surface, and expires
when the capture session ends.

## References & acknowledgements
* [Screen Capture](https://w3c.github.io/mediacapture-screen-share/)
* [Window Management](https://w3c.github.io/window-management)
* [Media Capture and Streams](https://w3c.github.io/mediacapture-main/)
