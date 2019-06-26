# WebXR Device API - Input
This document explains the portion of the WebXR APIs for managing input across the range of XR hardware. For context, it may be helpful to have first read about [WebXR Session Establishment](explainer.md) and [Spatial Tracking](spatial-tracking-explainer.md). Further information can also be found in the [Hit Testing explainer](hit-testing-explainer.md).

## Concepts
In addition to the diversity of tracking and display technology, XR hardware may support a wide variety of input mechanisms including screen taps, motion controllers (with multiple buttons, joysticks, triggers, touchpads, etc), voice commands, spatially-tracked articulated hands, single button clickers, and more. Despite this variation, all XR input mechanisms have a common purpose: enabling users to aim in 3D space and perform an action on the target of that aim. This concept is known as "target and select" and is the foundation for how input is exposed in WebXR.

### Targeting categories
All WebXR input sources can be divided into one of three categories based on the method by which users must target: 'gaze', 'tracked-pointer', and 'screen'.

#### Gaze
Gaze-based input sources do not have their own tracking mechanism and instead use the viewer's head position for targeting. Example include 0DOF clickers, headset buttons, regular gamepads, and certain voice commands. Within this category, some input sources are persistent (e.g. those backed by hardware) while others will come-and-go when invoked by the user (e.g. voice commands).

#### Tracked Pointer
Tracked pointers are input sources able to be tracked separately from the viewer. Examples include the Oculus Touch motion controllers and the Magic Leap hand tracking. For motion controllers, the target ray will often have an origin at the tip of motion controller and be angled slightly downward for comfort. The exact orientation of the ray relative to a given device follows platform-specific guidelines if there are any. In the absence of platform-specific guidance or a physical device, the target ray points in the same direction as the user's index finger if it was outstretched. Within this category, input sources are considered connected even if they are temporarily unable to be tracked in space.

#### Screen
Screen based input is driven by mouse and touch interactions on a 2D screen that are then translated into a 3D targeting ray. The targeting ray originates at the interacted point on the screen as mapped into the input `XRSpace` and extends out into the scene along a line from the screen's viewer pose position through that point. The specific mapped depth of the origin point depends on the user agent. It SHOULD correspond to the actual 3D position of the point on the screen where available, but MAY also be projected onto the closest clipping plane (defined by the smaller of the `depthNear` and `depthFar` attributes of the `XRSession`) if the actual screen placement is not known.

To accomplish this, pointer events over the relevant screen regions are monitored and temporary input sources are generated in response to allow unified input handling. For inline sessions the monitored region is the canvas associated with the `baseLayer`. For immersive sessions (e.g. hand-held AR), the entire screen is monitored. 

### Selection styles
In addition to a targeting ray, all input sources provide a mechanism for the user to perform a "select" action. This user intent is communicated to developers through events which are discussed in detail in the [Input events](#input-events) section. The physical action which triggers this selection will differ based on the input type. For example (though this is hardly conclusive):

  * Pressing a trigger
  * Clicking a touchpad
  * Tapping a button
  * Making a hand gesture
  * Speaking a command
  * Clicking or touching the screen

## Basic usage

### Enumerating input sources
The `inputSources` attribute on an `XRSession` returns a list of all `XRInputSource`s that the user agent considers active. The properties of an `XRInputSource` object are immutable. If a device can be manipulated in such a way that these properties can change, the `XRInputSource` will be removed from the array and a new entry created.

```js
let inputSources = xrSession.inputSources;
```

When input sources are added to or removed from the list of available input sources the `inputsourceschange` event must be fired on the `XRSession` object to indicate that any cached copies of the list should be refreshed. In addition, the `inputsourceschange` event will also fire once after the session creation callback completes. This event is of the type `XRInputSourceChangeEvent` and will contain three attributes: `session` is associated session being changed, `added` is the new input sources, and `removed` is the input sources that will no longer be reported.

```js
function onSessionStarted(session) {
  // Session initialization logic ...

  xrSession.addEventListener('inputsourceschange', onInputSourcesChange);

  // More session initialization logic ...
}

let xrInputSources = null;
function onInputSourcesChange(event) {
  xrInputSources = event.session.inputSources;
}
```

### Targeting ray pose
The `targetRaySpace` attribute of an `XRInputSource` is an `XRSpace` representing the inputs source's targeting ray origin and direction in space. (For more information on `XRSpace`s, see the [Spatial Relationships](spatial-tracking-explainer.md#spatial-relationships) section in the Spatial Tracking explainer). All `XRInputSource` objects, regardless of the `targetRayMode`, will have a valid `targetRaySpace`. As mentioned in the [Targeting Categories](#targeting-categories) section, the location of this `targetRaySpace` will vary based on the `targetRayMode`. For example, all `XRInputSource`s with a `targetRayMode` of 'gaze' will have the same `XRSpace` object as the `targetRaySpace`. This common `targetRaySpace` will represent the same location as the `XRSession.viewerSpace`, but will be a different object in order to keep the API flexible enough to add eye tracking in the future. Alternatively, the `targetRaySpace` of an input source with the `targetRayMode` of `tracked-pointer` will be based on the spatial location of the physical input device.

Th location of the `targetRaySpace` can be determined for a given frame by passing it to `XRFrame.getPose()` as the `space` parameter. Developers will likely supply their active `XRReferenceSpace` as the `relativeTo` parameter so they can use a consistent coordinate system, though this is not required. The result from `getPose()` should always be verified as the function may return `null` in cases where tracking has been lost or the `XRInputSource` instance is no longer connected or available.

```js
let inputSourcePose = xrFrame.getPose(inputSource.targetRaySpace, xrReferenceSpace);
if (inputSourcePose) {
  // do something with the result
}
```

The `XRPose` object has an `emulatedPosition` property that is used to indicate when the position components of a pose are not based on sensor data. This is the case, for example, for the Daydream and GearVR 3DoF controllers, which use an arm model to approximate controller position based on rotation. It may also be true on devices while under tracking loss. In these situations, the `emulatedPosition` attribute of the `XRPose` returned by `XRFrame.getPose()` will be set to `true` to indicate that the translation components of retrieved pose matrices may not be accurate.

## Input events
When the selection mechanism for an `XRInputSource` is invoked, three `XRInputSourceEvent` events are fired on the `XRSession`.

* `selectstart` indicates an action has been initiated. Example actions that fire this event are pressing a button or trigger.
* `selectend` indicates an action has ended. Example actions that fire this event are releasing a button or trigger. A `selectend` event must also be fired if the input source is disconnected after an action has been initiated, or the action has otherwise been cancelled. In that case an associated `select` event will not be fired.
* `select` indicates that a action has been completed. A `select` event is considered to be [triggered by user activation](https://html.spec.whatwg.org/multipage/interaction.html#triggered-by-user-activation)

The `selectstart` and `selectend` events are useful for handling dragging, painting, or other continuous motions. As the `select` event is trigger by user activation, it can be used to begin playing media or other trusted interactions.

```js
function onSessionStarted(session) {
  // Session initialization logic ...

  session.addEventListener("select", onSelect);
  session.addEventListener("selectstart", onSelectStart);
  session.addEventListener("selectend", onSelectEnd);

  // More session initialization logic ...
}
```

All three events are `XRInputSourceEvent` events. When fired the event's `inputSource` attribute will contain the `XRInputSource` that produced the event. The event's `frame` attribute will contain a valid `XRFrame` that can be used to call `getPose()` at the time the selection event occurred. Calling the frame's `getViewerPose()` method will throw an `InvalidState` error. (The viewer's pose can still be queried by passing an `XRReferenceSpace` of type `viewer` to `XRFrame.getPose()`.)

### Transient input sources
Some input sources are only be added to the list of input sources while an action is occurring. For example, those with an `targetRayMode` of 'screen' or those with `targetRayMode` of 'gaze' which are triggered by a voice command. In these cases, `XRInputSource` is only present in the `inputSources` array during the lifetime of the action. In this circumstance, the order of events is as follows:
1. Optionally `pointerdown`
1. `inputsourceschange` for add
1. `selectstart`
1. Optionally `pointermove`
1. `select`
1. Optionally `click`
1. `selectend`
1. `inputsourceschange` for remove
1. Optionally `pointerup`

Input sources that are instantaneous, without a clear start and end point such as a verbal command like "Select", will still fire all events in the sequence.

### Choosing a preferred input source
Many platforms support multiple input sources concurrently. Examples of this are left/right handed motion controllers or hand tracking combined with a 0DOF clicker. Since `xrSession.inputSources` returns all connected input sources, an application may choose to take into consideration the most recently used input sources when rendering UI hints, such as a cursor, ray or highlight.

To simplify the sample code throughout this explainer, an example is provided which shows one potential way to set a `preferredInputSource`.

```js
// Keep track of the preferred input source
var preferredInputSource = null;

function onSelectStart(event) {
    // Update the preferred input source to be the last one the user interacted with
    preferredInputSource = event.inputSource;
}

function onInputSourceChanged(event) {
  xrInputSources = event.session.inputSources;

  // Choose an appropriate default from available inputSources, such as 
  // prioritizing based on the value of targetRayMode: 'screen' over 
  // 'tracked-pointer' over 'gaze'.
  preferredInputSource = computePreferredInputSource();
}
```

## Rendering Input
When rendering an XR scene, it is often useful for users to have visual representations of their input sources and visual targeting hints. The mechanisms for displaying this information are often categorized as follows:
* **Highlight** a change in a virtual object's visualization that indicates it is being targeted
* **Cursor** a mark at the intersection point of an input source's targeting ray with 3D geometry
* **Pointing ray** a line drawn from the origin of an `XRInputSource.targetRaySpace` that terminates at the intersection with virtual or real geometry
* **Renderable model** a renderable virtual representation of a physical `XRInputSource`

The appropriateness of using these visualizations depends on various factors, including the value of the input source's `targetRayMode`.

|                   | Highlight | Cursor | Pointing Ray | Renderable Model |
| ------------------| --------- | ------ | ------------ | ---------------- |
| 'screen'          | √         | X      | X            | X                |
| 'gaze'            | √         | √      | X            | X                |
| 'tracked-pointer' | √         | √      | √            | √ (if possible)  |

There are several points worth calling out about the table above. First, 'screen' style inputs should only use highlights as the user's fingers will obscure other visualizations. Second, a pointing ray should not be drawn for 'gaze' style input sources because the ray's origin would be located between the user's eyes and may obscure the user's vision or be difficult to visually converge on. Third, developers should only attempt to draw renderable models and pointing rays for 'tracked-pointer' input sources, a topic explained in more detail in the [Renderable models](#renderable-models) section.

### Visualizing targeting hints
In order to draw targeting hints such as cursors, highlights, and pointing rays, a hit test must be performed against the 3D geometry to find what the user is aiming at. There are two types of hit testing: virtual and real-world. The sample code in this explainer will focus on virtual hit testing. For more information on real-world hit testing and how to combine it with virtual hit testing, see the [Hit Testing explainer](hit-testing-explainer.md).

WebXR does not have any knowledge of the developer's 3D scene graph, but does have information about the real-world location of `XRInputSource` objects. Using the `XRFrame.getPose()` function, as described in the [Targeting ray pose](#targeting-ray-pose) section, developers can determine position and orientation of the `XRInputSource`'s targeting ray and pass it into their 3D engine's virtual hit test function.

```js
function updateScene(timestamp, xrFrame) {
  // Scene update logic ...

  // Use the previously determined preferredInputSource to hit test with
  let inputSourcePose = xrFrame.getPose(preferredInputSource.targetRaySpace, xrReferenceSpace);
  if (inputSourcePose) {
    // Invoke the example 3D engine to compute a virtual hit test
    var virtualHitTestResult = scene.virtualHitTest(new XRRay(inputSourcePose.transform));
  }

  updateCursor(virtualHitTestResult);
  updateHighlight(virtualHitTestResult);
  updateRenderableInputModels(xrFrame);
  updatePointingRay(inputSourcePose, virtualHitTestResult);
  
  // Other scene update logic ...
}
```

#### Cursors
In the sample code below, a cursor is positioned based on the virtual hit-test result from the 3D engine's `preferredInputSource`. If an intersection doesn't exist or the `targetRayMode` of the `preferredInputSource` is 'screen', the cursor is hidden.

```js
function updateCursor(virtualHitTestResult) {
  // Toggle the cursor in the imaginary 3D engine
  if (!virtualHitTestResult || preferredInputSource.targetRayMode == "screen") {
    scene.cursor.visible = false;
  } else {
    scene.cursor.visible = true;
    scene.cursor.setTransform(virtualHitTestResult.transform);
  }
}
```

#### Highlights
In the sample code below, if the virtual hit test intersected with a virtual object the object is decorated with a highlight. Otherwise, the highlight is cleared.

```js
function updateTargetHighlight(virtualHitTestResult) {
  // The virtualTarget object isn't part of the WebXR API. It is
  // something set by the imaginary 3D engine in this example
  if (virtualHitTestResult && virtualHitTestResult.objectHit) {
    scene.addHighlight(virtualHitTestResult.objectHit);
  } else {
    scene.addHighlight(null);
  }
}
```

#### Pointing rays
In the sample below, a pointing ray is drawn for `XRInputSource` objects with the `targetRayMode` of 'tracked-pointer'. The ray starts at the origin of the `targetRaySpace` and terminates at the point of intersection with the 3D geometry, if one is available. If no intersection is found, a default ray length is used.

```js
function updatePointingRay(inputSourcePose, virtualHitTestResult) {
  // Toggle the pointing ray in the imaginary 3D engine
  if (!inputSourcePose || preferredInputSource.targetRayMode != "tracked-pointer") {
    scene.pointingRay.visible = false;
  } else {
    scene.pointingRay.visible = true;
    scene.pointingRay.setTransform(inputSourcePose.transform);
    if (virtualHitTestResult) {
      scene.pointingRay.length = MatrixMathLibrary.distance(inputSourcePose.transform, virtualHitTestResult.transform);
    } else {
      scene.pointingRay.length = scene.pointingRay.defaultLength;
    }
  }
}
```

### Renderable models
For `tracked-pointer` input sources, it is often appropriate for the application to render a contextually appropriate model (such as a racket in a tennis game). Other times it's desirable for the application to render a device that matches what the user is holding, especially when relaying instructions about it's use. Finally, there are times when it's best to not render anything at all, such as when the XR device uses a transparent display and the user can see their hands and/or any tracked devices without app assistance. See [Handling non-opaque displays](explainer.md#Handling-non-opaque-displays) in the main explainer for more details.


#### Choosing renderable models
The majority of `tracked-pointer` input sources will have a non-null `gamepad` attribute on the `XRInputSource` object. The `Gamepad`'s `id` is used to determine what should be rendered if the app intends to visualize the input source itself, rather than an alternative virtual object. (See the section on [Button and Axis State](#button-and-axis-state) for more details.)

The WebXR Device API currently does not offer any way to retrieve renderable resources that represent the input devices from the API itself, and as such the `XRInputSource`'s `profiles` must be used to identify and load an appropriate resources. (For example, from the application's server, a CDN, or a local cache.) The `profiles` provides a list of strings that identify the device, given in descending order of preference.

For example, the Samsung Odyssey controller is a variant of the standard Windows Mixed Reality controller. As a result, the `profiles` for that controller could be:

```js
// Exact strings are examples only.
["samsung-odyssey", "windows-mixed-reality", "touchpad-thumbstick-controller"]
```

Applications should iterate through the list until a string is located that corresponds to a known model, which should then be used when rendering the input device. The example below presumes that the `getInputSourceRenderableModel` call would do the required lookup and caching.

```js
function loadRenderableInputModels(xrInputSource) {
  for (let profile of xrInputSource.profiles) {
    // Retrieve a mesh to render based on the gamepad object's profile and handedness
    let renderableModel = getInputSourceRenderableModel(profile, xrInputSource.handedness);
    
    if (renderableModel) {
      // Add the model to the imaginary 3D engine's scene.
      scene.inputObjects.add(renderableModel, xrInputSource);
      return;
    }
  }

  // If the profiles list was empty or a corresponding model could not be found
  // for any entry in it the application could respond by not rendering the
  // device at all or rendering a generic device that is not intended to be a
  // visual match. This sample chooses the latter approach.
  scene.inputObjects.add(getDefaultInputSourceRenderableModel(), xrInputSource);
}
```

#### Placing renderable models
The `targetRaySpace` should not be used to place the renderable model of a 'tracked-pointer'. Instead, 'tracked-pointer' input sources will have a non-null `gripSpace` which should be used instead. The `gripSpace` is an `XRSpace` where, if the user was holding a straight rod in their hand, it would be aligned with the negative Z axis (forward) and the origin rests at their palm. In many cases this will be different from the `targetRaySpace` such as on a motion controller with a tip angled slightly downward for comfort.

Using the `gripSpace` developers can properly render a virtual object held in the user's hand. This could be something like a virtual sword positioned so that the blade points directly down the negative Z axis and the center of the handle is at the origin.

Similar to the description in the [Targeting ray pose](#targeting-ray-pose) section, developers should pass their `gripSpace` to `XRFrame.getPose()` each frame for updated location information. Developers should take care to check the result from `getPose()` as it may return `null` in cases where tracking has been lost or the `XRSpace`'s `XRInputSource` instance is no longer connected or available.

Finally, rendered models of input sources should generally only be shown if the `XRSession`'s `environmentBlendMode` is `'opaque'`, as any other mode implies that the user can see any physical device they may be holding. Rendering cursors, highlights, and pointing rays may still be desirable, depending on the application's needs.

```js
function updateRenderableInputModels(xrFrame) {
  // Don't display input models if the blend mode is not 'opaque'
  if (xrFrame.session.environmentBlendMode != 'opaque')
    return;

  foreach(inputObject of scene.inputObjects) {
    let xrInputSource = inputObject.xrInputSource;
    if(xrInputSource.gripSpace) {
      let pose = xrFrame.getPose(xrInputSource.gripSpace, xrReferenceFrame);
      if (pose) {
        inputObject.setTransform(pose.transform);
        inputObject.visible = true;
      } else {
        inputObject.visible = false;
      }
    }
  }
}
```

## Button and Axis State

Some applications need more than point-and-click style interaction provided by the `select` events. For input sources with buttons and axes, more complete information about the state of those inputs can be observed via the `XRInputSource`'s `gamepad` attribute. `gamepad` is an instance of the [`Gamepad`](https://w3c.github.io/gamepad/#gamepad-interface) interface if the input source has buttons and axes to report, and `null` otherwise.

Examples of input sources that may expose their state this way include Oculus Touch, Vive wands, Oculus Go and Daydream controllers, or other similar devices. Input devices not directly associated with the XR device, such as the majority of traditional gamepads, and tracked devices without discreet inputs, such as optical hand tracking, must not be exposed using this interface. 

`Gamepad` instances reported in this way have several notable behavioral changes vs. the ones reported by `navigator.getGamepads()`:

  - `Gamepad` instances connected to an `XRInputSource` must not be included in the array returned by `navigator.getGamepads()`.
  - The `Gamepad`'s `id` attribute must be `""` (empty string).
  - The `Gamepad`'s `index` attribute must be `-1`.
  - The `Gamepad`'s `connected` attribute must be `true` unless the related `XRInputSource` is removed from the `inputSources` array or the related `XRSession` is ended.

The exact button and axes layout is given by the `XRInputSource`'s `profiles` attribute, which contains an array of strings that identify the button and axes layout or subsets of it, ordered from most specific to least specific.

```js
function onXRFrame(timestamp, frame) {
  let inputSource = primaryInputSource;

  // Check to see if the input source has gamepad data.
  if (inputSource && inputSource.gamepad) {
    let gamepad = inputSource.gamepad;
    
    // Use joystick or touchpad values for movement.
    if (gamepad.axes.length >= 2) {
      MoveUser(gamepad.axes[0], gamepad.axes[1]);
    }

    // If the first gamepad button is pressed, perform an action.
    if (gamepad.buttons.length >= 1 && gamepad.buttons[0].pressed) {
      EmitPaint();
    }
    
    // etc.
  }

  // Do the rest of typical frame processing...
}
```

If the application includes interactions that require user activation (such as starting media playback), the application can listen to the `XRInputSource`s `select` events, which fire for every pressed and released button on the controller. When triggered by a controller input, the `XRInputSourceEvent` will include a `buttonIndex` other than `-1` to indicate which button on the gamepad triggered the event.

The UA may update the `gamepad` state at any point, but it must remain constant while running a batch of `XRSession` `requestAnimationFrame` callbacks or event callbacks which provide an `XRFrame`.

### XR gamepad mapping

The WebXR Device API also introduces a new standard controller layout indicated by the `mapping` value of `xr-standard`. (Additional mapping variants may be added in the future if necessary.) This defines a specific layout for the inputs most commonly found on XR controller devices today. The following table describes the buttons/axes and their physical locations:

| Button     | `xr-standard` Location            |
| ---------- | --------------------------------- |
| buttons[0] | Primary trigger                   |
| buttons[1] | Primary Touchpad/Joystick click   |
| buttons[2] | Grip/Secondary trigger            |
| buttons[3] | Secondary Touchpad/Joystick click |

| Axis    | `xr-standard` Location        |
| ------- | ----------------------------- |
| axes[0] | Primary Touchpad/Joystick X   |
| axes[1] | Primary Touchpad/Joystick Y   |
| axes[2] | Secondary Touchpad/Joystick X |
| axes[3] | Secondary Touchpad/Joystick Y |

Additional device-specific inputs may be exposed after these reserved indices, but devices that lack one of the canonical inputs must still preserve their place in the array. If a device has both a touchpad and a joystick the UA should designate one of them to be the primary axis-based input and expose the other at axes[2] and axes[3] with an associated button at button[3].

In order to make use of the `xr-standard` mapping, a device must meet **at least** the following criteria:

 - Is a `tracked-pointer` device. 
 - Has a trigger or similarly accessed button
 - Has at least one touchpad or joystick

devices that lack one of those elements may still expose `gamepad` data, but must not claim the `xr-standard` mapping. For example: The controls on the side of a Gear VR would not qualify for the `xr-standard` mapping because they represent a `gaze`-style input. Similarly, a Daydream controller would not qualify for the `xr-standard` mapping since it lacks a trigger.

### Exposing button/axis values with action maps

Some native APIs rely on what's commonly referred to as an "action mapping" system to handle controller input. In action map systems the developer creates a list of application-specific actions (such as "undo" or "jump") and suggested input bindings (like "left hand touchpad") that should trigger the related action. Such systems may allow users to re-bind the inputs associated with each action, and may not provide a mechanism for enumerating or monitoring the inputs outside of the action map.

When using an API that limits reading controller input to use of an action map, it is suggested that a mapping be created with one action per possible input, given the same name as the target input. For example, an similar mapping to the following may be used for each device:

| Button/Axis | Action name      | Sample binding            |
|-------------|------------------|---------------------------|
| button[0]   | "trigger"        | "[device]/trigger"        |
| button[1]   | "touchpad-click" | "[device]/touchpad/click" |
| button[2]   | "grip"           | "[device]/grip"           |
| axis[0]     | "touchpad-x"     | "[device]/touchpad/x"     |
| axis[1]     | "touchpad-y"     | "[device]/touchpad/y"     |

If the API does not provided a way to enumerate the available input devices, the UA should provide bindings for the left and right hand instead of a specific device and expose a `Gamepad` for any hand that has at least one non-`null` input.

The UA must not make any attempt to circumvent user remapping of the inputs.

### Generic Profiles

The strings returned in the `profiles` array of an `XRInputSource` describe the device being used with varying levels of detail, ranging from exactly identifying the device to only giving a broad description of it's shape and capabilities. It's highly recommended that, when applicable, the last profile in the array be from a list of well-known "generic" profiles, given below.

 - **"button-controller":** A controller with at least one button/trigger but no touchpad or thumbstick. Controllers with this profile must not use the `xr-standard` Gamepad mapping.
 - **"touchpad-controller"** A controller with a touchpad, but no thumbstick. If the controller also has at least one additional button or trigger it must use the `xr-standard` Gamepad mapping.
 - **"thumbstick-controller"** A controller with a thumbstick, but no touchpad. If the controller also has at least one additional button or trigger it must use the `xr-standard` Gamepad mapping.
 - **"touchpad-thumbstick-controller"** A controller with both a touchpad and a thumbstick. If the controller also has at least one additional button or trigger it must use the `xr-standard` Gamepad mapping.

More generic profiles may be added to this list over time as new common form factors are observed.

## Appendix A: Proposed partial IDL
This is a partial IDL and is considered additive to the core IDL found in the main [explainer](explainer.md).
```webidl
//
// Session
//

partial interface XRSession {
  readonly attribute XRInputSourceArray inputSources;
  
  attribute EventHandler onselect;
  attribute EventHandler onselectstart;
  attribute EventHandler onselectend;
  attribute EventHandler oninputsourceschange;
};

//
// Input
//

enum XRHandedness {
  "none",
  "left",
  "right"
};

enum XRTargetRayMode {
  "gaze",
  "tracked-pointer",
  "screen"
};

[SecureContext, Exposed=Window]
interface XRInputSource {
  readonly attribute XRHandedness handedness;
  readonly attribute XRTargetRayMode targetRayMode;
  readonly attribute XRSpace targetRaySpace;
  readonly attribute XRSpace? gripSpace;
  readonly attribute Gamepad? gamepad;
  readonly attribute FrozenArray<DOMString> profiles;
};

[SecureContext, Exposed=Window]
interface XRInputSourceArray {
  iterable<XRInputSource>;
  readonly attribute unsigned long length;
  getter XRInputSource(unsigned long index);
};

//
// Events
//

[SecureContext, Exposed=Window, Constructor(DOMString type, XRSessionEventInit eventInitDict)]
interface XRSessionEvent : Event {
  readonly attribute XRSession session;
};

dictionary XRSessionEventInit : EventInit {
  required XRSession session;
};

[SecureContext, Exposed=Window, Constructor(DOMString type, XRInputSourceChangeEventInit eventInitDict)]
interface XRInputSourceChangeEvent : Event {
  readonly attribute XRSession session;
  readonly attribute FrozenArray<XRInputSource> removed;
  readonly attribute FrozenArray<XRInputSource> added;
};

dictionary XRInputSourceChangeEventInit : EventInit {
  required XRSession session;
  required FrozenArray<XRInputSource> removed;
  required FrozenArray<XRInputSource> added;
};

[SecureContext, Exposed=Window,
 Constructor(DOMString type, XRInputSourceEventInit eventInitDict)]
interface XRInputSourceEvent : Event {
  readonly attribute XRFrame frame;
  readonly attribute XRInputSource inputSource;
  readonly attribute long buttonIndex;
};

dictionary XRInputSourceEventInit : EventInit {
  required XRFrame frame;
  required XRInputSource inputSource;
  long buttonIndex = -1;
};
```
