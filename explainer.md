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

## Obtaining camera extrinsics and intrinsics

Some Computer Vision algorithms may depend on camera intrinsics and extrinsics in order to correctly perform. The camera extrinsics can be obtained by inspecting the `XRView`'s `transform` attribute (this is relative to the reference space used to obtain the viewer pose). The camera intrinsics can be computed given a projection matrix returned from an `XRView`, as well as `XRViewport`. Below, we will go through 2 different paths of calculating the transformation from camera space coordinates to screen space coordinates - one relying on projection matrix and viewport properties, the other relying on camera intrinsics matrix - and use the known parameters from the former approach to calculate the unknown parameters for latter calculation method.

**Note:** The steps to obtain camera intrinsics and extrinsics are only valid since the camera image returned by the API is perfectly aligned with the `XRView` from which it was obtained.

### Camera space to screen space - projection matrix and viewport route

Projection matrix convetions as per: http://www.songho.ca/opengl/gl_projectionmatrix.html.

Starting from a projection matrix with the following parameters:

```
P = p0  p4  p8   p12
    p1  p5  p9   p13
    p2  p6  p10  p14
    p3  p7  p11  p15
```

We can repeat the derivation of the projection matrix from [songho.ca](http://www.songho.ca/opengl/gl_projectionmatrix.html), allowing for non-zero skew:

```
P = p0  p4  p8   0    = 2n/(r-l)  skew      (r+l)/(r-l)  0
    0   p5  p9   0      0         2n/(t-b)  (t+b)/(t-b)  0
    0   0   p10  p14    0         0        -(f+n)/(f-n) -2fn/(f-n)
    0   0  -1    0      0         0        -1            0
```

The skew factor controls how much of the Y coordinate is mixed into the X coordinate. It is usually zero, but WebXR allows nonzero skew values which results in nonrectangular pixels.

A GL projection matrix transforms from camera space to clip space, then to NDC after perspective divide. This needs to be scaled to pixels based on the viewport `vp`. The NDC x and y ranges `(-1 .. 1)` are then transformed to `(vp.x .. vp.x + vp.width)` and `(vp.y .. vp.y + vp.height)`, respectively. For example, NDC x coordinate is transformed to screen space:

```
screen_x = vp.w * (ndc_x + 1)/2 + vp.x
         = (vp.w/2) * ndc_x + (vp.w/2 + vp.x)
```

Using a matrix S for the NDC-to-screen-coordinate transform described above, this becomes:

```
p_screen.xy = (S * p_ndc).xy

with S = vp.w/2  0       0  vp.w/2 + vp.x
         0       vp.h/2  0  vp.h/2 + vp.y
         0       0       1  0
         0       0       0  1
```

The camera-space point transformation into screen space is then as follows:

```
p_screen.xy = (S * p_ndc).xy
            = (S * p_clip).xy / p_clip.w
            = (S * P * p_camera).xy / (P * p_camera).w
            = (S * P * p_camera).xy / (-p_camera.z)
```

Note that this uses the usual GL convention of looking along the negative Z axis, with negative-z points being visible.

### Camera space to screen space - intrinsic matrix route

Intrinsic matrix convention as per https://en.wikipedia.org/wiki/Camera_resectioning#Intrinsic_parameters

The intrinsic matrix K transforms from camera space to homogenous screen space, providing pixel screen coordinates after the perspective divide. This convention assumes looking along the positive Z axis, with positive-z points being visible.

```
K = ax  gamma  u0  0
    0   ay     v0  0
    0   0      1   0
```

For compatibility with WebXR, insert a placeholder 3rd row to get a 4x4 matrix `Kexp` and invert the Z coordinate. This produces a modified intrinsic matrix K':

```
K' = 1  0  0  0 * Kexp = ax  gamma -u0  0
     0  1  0  0          0   ay    -v0  0
     0  0 -1  0          *   *      *   *
     0  0  0  1          0   0     -1   0
```

This results in the following transformation from camera space to screen space:

```
p_screen.xy = (K' * p_camera).xy / (K' * p_camera).w
            = (K' * p_camera).xy / (-p_camera.z)
```

### Putting it together

Since the `p_screen.xy` coordinates must be the same for both calculation methods, it
follows that the intrinsic matrix K' is S * P:

```
p_screen.xy = (K' * p_camera).xy / (-p_camera.z)
            = (S * P * p_camera).xy / (-p_camera.z)

=>
  K' = S * P
```

For example, K'[0,2] is -u0, and equals the product of row 0 of S with column 2 of P:

```
K'[0,2] = S[0,] * P[,2]
    -u0 = [vp.v/2, 0, 0, vp.w/2 + vp.x] * [p8, p9, p10, -1]
        = (vp.w/2) * p8 + 0 * p9 + 0 * p10 + (vp.w/2 + vp.x) * (-1)
        = vp.w/2 * (p8 - 1) - vp.x

=>
  u0 = vp.w/2 * (1 - p8) + vp.x
```

Repeating the calculation for other intrinsic matrix parameters, we arrive at the following function:

```javascript
function getCameraIntrinsics(projectionMatrix, viewport) {
  const p = projectionMatrix;

  // Principal point in pixels (typically at or near the center of the viewport)
  let u0 = (1 - p[8]) * viewport.width / 2 + viewport.x;
  let v0 = (1 - p[9]) * viewport.height / 2 + viewport.y;

  // Focal lengths in pixels (these are equal for square pixels)
  let ax = viewport.width / 2 * p[0];
  let ay = viewport.height / 2 * p[5];

  // Skew factor in pixels (nonzero for rhomboid pixels)
  let gamma = viewport.width / 2 * p[4];

  // Print the calculated intrinsics:
  const intrinsicString = (
    "intrinsics: u0=" + u0 + " v0=" + v0 + " ax=" + ax + " ay=" + ay +
    " gamma=" + gamma + " for viewport {width=" +
    viewport.width + ",height=" + viewport.height + ",x=" +
    viewport.x + ",y=" + viewport.y + "}");

  console.log("projection:", Array.from(projectionMatrix).join(", "));
  console.log(intrinsicString);
}
```

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
