# Prelude
For a version of this that is more comprehensive and easier to navigate thanks to internal links, please refer to [the explainer](https://docs.google.com/document/d/1oSDmBPYVlxFJxb7ZB_rV6yaAaYIBFDphbkx5bXLnzFg/edit?usp=sharing).

# Problem Description

## Generic Problem Description

Consider a web-application, running in one tab, which we’ll name “main_app.” Assume main_app calls [getDisplayMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia) and the user chooses to share another tab, where an application is running which we’ll call “captured_app.”

Note that:

1. main_app does not know what it is capturing.
2. captured_app does not know that it is being captured; let alone by whom.

Both these traits are desirable for the general case, but there exist legitimate use cases where the browser would want to allow applications to opt-in to bridging that gap and enable a connection.

We wish to enable the legitimate use cases while keeping the general case as it was before.

## Use-case #1: Cross-App Communications

Consider two applications that wish to cooperate, for example a VC app and a presentation app. Assume the user is in a VC session. The user starts sharing a presentation. Both applications are interested in letting the VC app discover that it is capturing a slides session, which application, and even which session, so that the VC application will be able to expose controls to the user for flipping through slides. When the user clicks those controls, the VC app will be able to send messages to the presentation app (either through a service worker or through a shared back-end infrastructure). These messages will instruct the presentation app to flip through slides, enter/leave presentation-mode, etc.

## Use-case #2: Analytics

Capturing applications often wish to gather statistics over what applications their users tend to capture. For example, VC applications would like to know how often their users share presentation applications from specific providers, Wikipedia, CNN, etc. Gathering such information can be used to improve service for the users by introducing new collaborations, such as the one described above.

## Use-case #3: Detecting Unintended or Unapproved Captures

Users sometimes choose to share the wrong tab. Sometimes they switch to sharing the wrong tab by clicking the share-this-tab-insead button by mistake. A benevolent application could try to protect the user by presenting an in-app dialog for re-confirmation, if they believe that the user may have made a mistake.

## Use-case #4: Avoiding “Hall of Mirrors”

This use-case is a sub-case of #3, but deserves its own section due to its importance. The “Hall of Mirrors” effect occurs when users choose to share the tab in which the VC call takes place. When detecting self-capture, a VC application can avoid displaying the captured stream back to the user, thereby avoiding the dreaded effect.

# Our Solution

## Summary

* Captured applications opt-in to exposing information by setting CaptureHandleConfig.
* Capturing applications read this information as CaptureHandle, which is available through [two access points](https://docs.google.com/document/d/1oSDmBPYVlxFJxb7ZB_rV6yaAaYIBFDphbkx5bXLnzFg/edit#bookmark=kix.y4ww0mbjjv2x).

## MediaDevices.setCaptureHandleConfig
```
dictionary CaptureHandleConfig {
  boolean exposeOrigin = false;
  DOMString handle = “”;
  sequence<DOMString> permittedOrigins = [];
};

partial interface MediaDevices {
  void setCaptureHandleConfig(
      optional CaptureHandleConfig config = {});
};
```
We add the method setCaptureHandleConfig in [MediaDevices](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices). It accepts a configuration consisting of three independent members.

* **exposeOrigin:** If an application sets this value to true, the origin of that application may be exposed to capturing applications as CaptureHandle.origin.
* **handle:** If an application sets this value, that value is exposed to capturing applications as CaptureHandle.handle. Otherwise, the capturing application will see the empty string in that field.
Values to this field are limited to 1024 unicode code points. If the application attempts to set a longer value, a TypeError exception is raised.
* **permittedOrigins:** A capturing application is only allowed to observe CaptureHandle if its origin is included in permittedOrigins. If permittedOrigins includes “*”, all capturers are permitted to observe CaptureHandle.
In either of these cases, *origin* and *handle* are still exposed independently. That means that if *exposeOrigin* is false, capturers only see the handle.
Defaulting to the empty set means that by default, nobody will see the capture handle, and the call will have no effect. This ensures that the caller has made an explicit decision on which origins to expose the capture handle to.

### Calls from an Embedded Frame

When *setCaptureHandleConfig()* is called from a document which is not the top-level document, an error is thrown.

### Side Effects

Whenever a captured application calls *setCaptureHandleConfig()*:

1. An event is fired on the capturer side. It has the new CaptureHandle as a property.
2. Any subsequent calls to [MediaStreamTrack.getSettings()](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack/getSettings) on the relevant tracks will produce a new [MediaTrackSettings](https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackSettings) object with a new CaptureHandle object.

### Empty CaptureHandleConfig

To clarify, the empty CaptureHandleConfig is the one which has all values set to their defaults.

## The CaptureHandle Type
```
dictionary CaptureHandle {
  DOMString origin;
  DOMString handle;
};
```

CaptureHandle is the object through which a capturing application may read information about the application it is capturing. It contains the following independent fields:

* **origin:** If the captured application opted-in to exposing its origin (by setting *CaptureHandleConfig.exposeOrigin* to true), then *CaptureHandle.origin* is set to the origin of the captured application. Otherwise, the CaptureHandle.origin is not set.
* **handle:** Reflects the value which the captured app set in *CaptureHandleConfig.handle*.

Capturing applications have two points of access to CaptureHandle objects:

1. Through *MediaTrackSettings.captureHandle*.
2. Through *CaptureHandleUpdateEvent*.

To clarify, the empty CaptureHandle is defined as that where both origin and handle are set to the empty string. If the captured application sets the empty CaptureHandleConfig, then the capturing application will read the empty CaptureHandle.

## MediaTrackSettings.captureHandle
```
partial dictionary MediaTrackSettings {
  CaptureHandle captureHandle;
};
```

Assume capturer is the application in the current tab, and captured is an application running in another tab. Assume capturer is display-capturing the tab in which captured lives, and that track, of type [MediaStreamTrack](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack), is either a video or an audio track associated with this capture. Calling track.[getSettings()](https://developer.mozilla.org/en-US/docs/Web/API/MediaStreamTrack/getSettings) returns a [MediaTrackSettings](https://developer.mozilla.org/en-US/docs/Web/API/MediaTrackSettings) object with a new field, captureHandle, of type *CaptureHandle*.

## CaptureHandleUpdateEvent
```
[Exposed=Window]
interface CaptureHandleUpdateEvent : Event {
  constructor(CaptureHandleUpdateEventInit);
  [SameObject] readonly CaptureHandle captureHandle;
};
```

A CaptureHandleUpdateEvent is fired in the capturing application’s JS context whenever the CaptureHandleConfig is updated in the captured application:

* When the captured application calls *MediaDevices.setCaptureHandleConfig* and sets a new configuration.
* When the captured tab’s top-level application is navigated away from a site that has set a non-empty capture handle.
* If the user manually changes the captured-tab, assuming the new site has a different CaptureHande (as observable by the capturing app).

The event has a single property - captureHandle - containing the capture-handle as it was at the time the event was fired. (This in contrast to the current handle, which is accessible via *getSettings*. When multiple events are fired at rapid succession, each will contain its respective value, and only the last one can be guaranteed to be equal to that exposed by getSettings.)

If an application tries registering an event handler on a track that’s originating from the browsing context in which the application is running, an error should be raised and the handler value should remain unchanged.

CaptureHandleUpdateEventInit is basically a replica of CaptureHandle, as is the pattern for event constructors in WebIDL.

## MediaStreamTrack.oncapturehandlechange
```
partial interface MediaStreamTrack {
  attribute EventHandler oncapturehandleupdate;
};
```

Allows capturing applications to register an event-handler. The handler is associated with a specific track, and therefore only receives the particular *event* associated with that track.

## CaptureHandle Asynchronicity

* MediaStreamTrack.getSettings().captureHandle returns the latest observable value.
* CaptureHandleUpdateEvent objects contain (as a property) the CaptureHandle at the time the event was fired.

## Navigation of the Captured Application’s Tab

When the captured tab’s top-level document is navigated cross-document, before navigation occurs, if a capture handle is set, the browser implicitly resets it. This fires an event.

Corollaries:

1. Assume capture begins of an application with handle=”a”. Assume navigation then unloads the application and replaces it with another application which sets handle=”b”. Two events will be fired. The first, upon navigating away, will be associated with an empty CaptureHandle. The second, once the new site loads and sets handle=”b”, will be associated with that new value.
2. Navigation away from a site that sets a CaptureHandle is detectable by the capturer.

Navigation away from a site that did not set a CaptureHandle is not detectable by the capturer. (More accurately - not more easily detectable than before.)

# Privacy + Security Considerations

Please refer to [the explainer](https://docs.google.com/document/d/1oSDmBPYVlxFJxb7ZB_rV6yaAaYIBFDphbkx5bXLnzFg/edit?usp=sharing).
