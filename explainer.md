# WebXR Raw Camera Access

## Overview

Currently, to protect user privacy, WebXR Device API does not provide a way to grant raw camera access to the sites. Additionally, alternative ways for sites to obtain raw camera access (`getUserMedia()` web API) are not going to provide the application pose-synchronized camera images that could be integrated with WebXR Device API. For some scenarios, this limitation may pose a significant barrier to adoption of WebXR. This is especially true given the fact that addition of new APIs takes time, thus stifling experimentation and prototyping that could happen purely in JavaScript.

Given the above, it is beneficial to provide the sites access to raw camera images directly through WebXR API. It does come with a potential risk related to the fact that the sites will be granted access to the images from the user's environment. To mitigate privacy aspects of providing such capability, implementers should ensure that appropriate user consent is collected before granting camera access to the site.

The work in this repository describes experiments that likely falls under the scope of immersive-web CG's [Computer Vision](https://github.com/immersive-web/computer-vision) repository.

## Use cases

Granting the camera access to the application could allow the applications to:
- Take a snapshot of the AR experience along with application-rendered content. This is mostly relevant for scenarios where the camera image serves as the background for application's content (for example handheld AR).
- Allow the application to overlay some render effects over the entire scene.
- Provide feedback to the user about the environment they are located in. This may be particularly useful for VR headsets with available cameras (for example WMR headsets and the Flashlight feature). Note: this use case is not addressed by the current proposal - see [Immersive Web CG discussions](#Immersive-Web-CG-discussions) section.
- Run custom computer vision algorithms on the data obtained from the camera texture. It may for example enable applications to semantically annotate regions of the image, for example to provide features related to accessibility.


## Proposed API shape

The raw camera image access API should seamlessly integrate with APIs already exposed by the WebXR Device API. The Web IDL for proposed API could look roughly as follows:

```webidl
partial interface XRView {
  // Non-null iff there exists an associated camera that perfectly aligns with the view:
  [SameObject] readonly attribute XRCamera? camera;
};

interface XRCamera {
  // Dimensions of the camera image:
  readonly attribute long width;
  readonly attribute long height;
};

partial interface XRWebGLBinding {
  // Access to the camera texture itself:
  WebGLTexture? getCameraImage(XRCamera camera);
};
```

This allows us to provide a time-indexed texture containing a camera image that is retrievable only when the XRFrame is considered active. The API should also be gated by the `“camera-access”` feature descriptor.

## Using the API

The applications could leverage the newly introduced feature as follows:

1. Create a session that supports accessing camera images:
```javascript
const session = await navigator.xr.requestSession(“immersive-ar”, {
  requiredFeatures: [“camera-access”]
});
```

If UA decides it needs to prompt the user for permission to use the camera, it can do so at this stage.

2. In requestAnimationFrame callback, the application can iterate over the views and react appropriately when it finds a view with a camera associated with it:

```javascript
// ... in rAFcb ...
let viewerPose = xrFrame.getViewerPose(xrRefSpace);
for (const view of viewerPose.views) {
  if (view.camera) {
    // ... handle a view that has a camera image ...
  }
}
```

3. Using the view that has a non-null `camera` attribute, the application can then query the `XRWebGLBinding` for the camera image texture:
```javascript
// ... in rAFcb ...
const cameraTexture = binding.getCameraImage(view.camera);
```

Side note: GL binding referred to in step 3. is an interface that is newly introduced in the [WebXR Layers Module](https://immersive-web.github.io/layers/#XRWebGLBindingtype). It can be [constructed](https://immersive-web.github.io/layers/#dom-xrwebglbinding-xrwebglbinding) given an `XRSession` and `XRWebGLRenderingContext`.

## Alternatives considered

### Initial API proposal

First variant of the API was described in bialpio@google.com's personal repository, used to implenent a PoC in Chrome for Android.

```webidl
partial interface XRWebGLBinding {
  WebGLTexture? getCameraImage(XRFrame frame, XRView view);
};

partial interface XRViewerPose {
  [SameObject] readonly attribute FrozenArray<XRView> cameraViews;
};
```

The initial proposal of the API used an `XRView` type as a handle that can then be used to query the camera image from the system. The camera intrinsics could be calculated by inspecting the projection matrix of the XRView. In case the system wanted to surface a camera image that is not aligned with any of the existing views, it would have to artificially create additional `XRView` instances.

This API shape was not very extensible as it did not offer a clear way of adding camera-specific properties - they would have to be added to an `XRView`. In the likely case the API would have to be extended to account for cameras that are not aligned with views, the API shape did not offer nice / obvious extension points.

### Immersive Web CG discussions

Initial IW CG discussions for the feature happened in bialpio@google.com's personal [repository](https://github.com/bialpio/webxr-raw-camera-access/issues/1) (currently archived), and caused the API shape to change. The [outcome](https://github.com/bialpio/webxr-raw-camera-access/issues/1#issuecomment-821531579) of the discussion was to pivot from attempting to provide a unified API shape to cater to both smartphone-focused (render effects) and <abbr title="head-mounted display">HMD</abbr>-focused (running custom computer vision algorithms) scenarios, into a simpler API shape that would work primarily for smartphone-specific scenarios. The API shape could later be extended with a variant that would take into account various platform constraints of HMDs - this variant would also be feasible to implement on smartphones. In addition, application authors could use the currently proposed API shape to run custom CV algorithms on the returned camera texture if they choose to, until a better-suited, cross-platform solution becomes available in WebXR. For more details, see the discussion linked above.
