# TL;DR - Demos
Two quick demos are available:
1. Demo for remotely controlling a presentation [here](https://wicg.github.io/capture-handle/demos/remote_control/capturer.html).
2. Demo for detecting self-capture [here](https://wicg.github.io/capture-handle/demos/self_capture_detection/index.html).

# Summary

Capture Handle is a mechanism that allows a display-capturing web-application to ergonomically and confidently identify the web-application it is display-capturing (provided that the captured application has opted-in). Such identification allows these two applications to collaborate in interesting ways.

For example, if a VC application is capturing a presentation, then the VC application can expose user-controls for previous/next-slide directly in the VC application. This lets the user navigate presentations without having to jump between the VC and presentation tabs.

# Problem Description

## Generic Problem Description

Consider a web-application, running in one tab, which we’ll name “main_app.” Assume main_app calls [getDisplayMedia ](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia) and the user chooses to share another tab, where an application is running which we’ll call “captured_app.”

Note that:

1. main_app does not know what it is capturing.
2. captured_app does not know that it is being captured; let alone by whom.

Both these traits are desirable for the general case, but there exist legitimate use cases where the browser would want to allow applications to opt-in to bridging that gap and enable a connection.

We wish to enable the legitimate use cases while keeping the general case as it was before.

## Use-case #1: Cross-App Communications

Consider two applications that wish to cooperate, for example a VC app and a presentation app. Assume the user is in a VC session. The user starts sharing a presentation. Both applications are interested in letting the VC app discover that it is capturing a slides session, which application, and even which session, so that the VC application will be able to expose controls to the user for flipping through slides. When the user clicks those controls, the VC app will be able to send messages to the presentation app (either through a service worker or through a shared back-end infrastructure). These messages will instruct the presentation app to flip through slides, enter/leave presentation-mode, etc.

## Use-case #2: Avoiding “Hall of Mirrors”

The “Hall of Mirrors” effect occurs when users choose to share the tab in which the VC call takes place. When detecting self-capture, a VC application can avoid displaying the captured stream back to the user, thereby avoiding the dreaded effect. Other mitigation strategies are also possible based on Capture Handle.

## Use-case #3: Detecting Unintended or Unapproved Captures

Users sometimes choose to share the wrong tab. Sometimes they switch to sharing the wrong tab by clicking the share-this-tab-insead button by mistake. A benevolent application could try to protect the user by presenting an in-app dialog for re-confirmation, if they believe that the user may have made a mistake.

## Use-case #4: Analytics

Capturing applications often wish to gather statistics over what applications their users tend to capture. For example, VC applications would like to know how often their users share presentation applications from specific providers, Wikipedia, CNN, etc. Gathering such information can be used to improve service for the users by introducing new collaborations, such as the one described above.


# Our Solution

## Summary

* Captured applications opt-in to exposing information by setting CaptureHandleConfig.
* Capturing applications read this information as CaptureHandle, which is available through two access points (discussed below).

## Captured Applications (e.g. presentation software)

Application can call a newly added method, `MediaDevices.setCaptureHandleConfig()`, and opt-in to exposing information to capturing applications. Two pieces of information may be exposed:
* `exposeOrigin`: The captured applications's origin. (This is a boolean. Origin exposure is mediated by the browser, meaning the origin cannot be spoofed.)
* `handle`: An arbitrary string carrying semantic meaning of the captured application's choosing.

## Capturing Applications (e.g. video-conferencing software)

Capturing applications can read information exposed to them by captured applications. The immediate value is exposed via `MediaStreamTrack.getCaptureHandle()`. An event handler for on-change events is exposed as `oncapturehandlechange` (also on `MediaStreamTrack`).

## Controlled Exposure

The captured application may control which capturing applications are allowed to see this exposed information using `permittedOrigins`. Only capturers allowlisted by `permittedOrigins` may read the captured application's information.

## Sample Usage

Consider the case of a conference call on **VC-MAX** where the local user chooses to present another tab with a slides deck by **Slides 3000**. 

The captured application, Slides 3000, is aware it could be captured:
```
function getlSessionId() {
  ...  // Returns some ID which is meaningful using loonyAPI.
}

function onPageLoaded() {
  ...
  setCaptureHandleConfig({
    exposeOrigin: true,
    handle: JSON.stringify({
      description: "See slides-3000.com for our API. Collaborations welcome!",
      protocol: "loonyAPI",
      version: "1.983",
      sessionId: getlSessionId(),
    }),
    permittedOrigins = ['*']
  });
  ...
}
```

The capturing application, VC-MAX, reads the capture-handle of the captured display-surface the user chose:
```
  function startCapture() {
    ...
    const stream = await navigator.mediaDevices.getDisplayMedia();
    const [track] = stream.getVideoTracks();
    if (track.getCaptureHandle) {  // Feature detection.
      // Subscribe to notifications of the capture-handle changing.
      track.oncapturehandlechange = (event) => {
        OnNewCaputreHandle(event.captureHandle());
      };
      // Read the current capture-handle.
      OnNewCaputreHandle(track.getCaptureHandle());
    }
    ...
  }

  function OnNewCaputreHandle(captureHandle) {
    if (captureHandle.origin == 'slides-3000.com') {
      const parsed = JSON.parse(captureHandle.handle);
      OnNewSlides300Session(parsed.protocol, parsed.version, parsed.sessionId);
    }
  }

  function OnNewSlides300Session(protocol, version, sessionId) {
    if (protocol != "loonyAPI" || version > "2.02") {
      return;
    }
    ExposeSlides300Controls(sessionId);  // Exposes prev/next buttons to the user.
  }
```

# Privacy + Security Considerations

## Opt-In
By default, nothing new is revealed.

## User-driven
The process is still user-driven, as [getDisplayMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getDisplayMedia) itself is user-driven - the user chooses what to share, as well as whether to share.

## Theoretically No-op
When users consent to a capture, they implicitly allow the capturing-app to receive any information that the captured-app wishes broadcast. Such information could be broadcast by embedding “magic pixels” in the captured-app, embedding QR codes in the video stream, etc.

The change described in this document is almost a no-op from the perspective of security and privacy. The word “almost” is necessary because previously the capturing-app would have to either take that information with a grain of salt, or validate it externally, e.g. by establishing contact with the collaborating app over some external medium. Now, at least one piece of information is mediated by the browser and therefore known to be non-spoofable - the origin.

## Captured-App Still Cannot Discover it is Being Captured
This change does not allow a captured application to discover when it is being captured, unless the capturing application sends it an out-of-band message to that effect. This would have been possible regardless of the change described in this document, and is therefore not a concern.

**Note:** The general concern with capturing applications discovering they’re being captured is that they could use that fact in order to censor their own content, limiting the user’s control and aggravating them. This concern does not apply to capturers who choose to inform the captured application of the capture. Such applications could either stop informing the captured application, after all. And the user is at the mercy of the capturing application to begin with, which could have chosen to never even start the capture.

## Controlled Exposure
One concern is that a capturer could misbehave when capturing specific origins. For example, when a VC application detects the user is capturing a competitor’s productivity suite, it could display ads for its own productivity suite. This concern is mitigated through the CaptureHandleConfig’s permittedOrigins field, which allows applications to control which origins may observe the CaptureHandle they set.

## App-controlled Opaqueness
Applications can use any CaptureHandle.handle they wish, and may also independently choose whether to expose their origin. The handle can either expose information according to some advertised format, or it can be completely opaque to anyone but a few privileged, collaborating apps.

For example, HypotheticalSite could make it widely known that their format is “HypotheticalSite:<rand_guid>”. This could then be used in tandem with some other API exposed by HypotheticalSite, such as an API for remotely controlling a playback (subject to access-control set by HypotheticalSite on their own API).

Continuing with the example above, it would also be possible for HypotheticalSite to set their format to <rand_guid>, with rand_guid as a 32-character hexadecimal string, and set exposeOrigin=false. It would then be difficult for arbitrary capturers to find out if they’re capturing a HypotheticalSite tab, since many different applications could follow that handle pattern. However, HypotheticalSite could give select collaborating applications access to a HypotheticalSite-operated API that checks whether a given GUID is a valid HypotheticalSite ID.

## Non-spoofable Origin
If the captured-application opts into exposing its origin, the capturing application gains access to a field that is non-spoofable. Claims made in the capture-handle can be trusted by the capturer if they are known to originate from a given origin. If an application chooses to share a handle but not the origin, on the contrary, the capturing application can either treat that information as suspect, or verify it in some external way before using it.

## Improvements over Steganography
As previously mentioned, applications could have previously used QR codes or [steganographic means](https://en.wikipedia.org/wiki/Steganography) to advertise some capture-handle. However, that would have been susceptible to interference from embedded frames, either intentionally or not. The capture handle mechanism, in contrast, is only accessible to the top-level document, and is safe from interference.

## Capturer Can Detect Navigation
* Assume EXP is a site exposing something - possibly the origin, possibly a handle, possibly both. When we want to denote two sites exposing different configs, we’ll name them EXP1 and EXP2. 
* Assume sites NOEXP, NOEXP1, NOEXP2, etc. never call setCaptureHandleConfig. (Recall that this is treated as implicitly calling setCaptureHandleConfig with the empty config.)

We distinguish these types navigation events:
1. NOEXP to EXP: Non-exposing site to exposing site.
2. EXP to NOEXP: Exposing site to non-exposing site.
3. EXP1 to EXP2: One exposing site to another, different exposing site.
4. EXP1 to EXP1*: One exposing site to another, but which sets the same configuration.
5. NOEXP1 to NOEXP2: One non-exposing site to another non-exposing site.

#1 partially reveals navigation. Depending on additional parameters, the capturer might have either full or partial certainty over whether navigation occurred, or whether EXP set a capture handle relatively late.

#2 and #3 make navigation unconcealable by definition.

#4 is similar to #2 in its first stage. Recall that when navigating away from EXP1 to EXP1*, the browser does not know if/when EXP1* will call setCaptureHandleConfig, and must treat navigation away from EXP1 as an implicit call to setCaptureHandleConfig with the empty config.

#5 We don’t fire an event in this case (rationale). The result of this decision is that a capturer can detect navigation away from an exposing site, but not navigation away from a non-exposing site.

## Excessive Events
It is possible for a captured application to “bombard” its capturer with events by repeatedly calling setCaptureHandleConfig. Note that this is true regardless of whether the event fires only on a new config, or whenever setCaptureHandleConfig is called, since the application can alternately set two different handles. This concern does not seem significant, as:
1. The captured application would be expending the same general amount of resources as it would be costing the capturing application. (Note that the case of multiple capturers is very rare in practice, and even then is limited to only a handful of capturers.)
2. The captured application would normally be carrying out the attack without knowing whether it is being captured, let alone knowing by whom. (The case where the capturer communicated its identity back to the captured application presumes a level of collaboration that makes such an attack unlikely, and at any rate - within the power of the capturing application to avoid.)

## Incognito Mode
Calls to setCaptureHandleConfig from an incognito tab must not be blocked, so as to avoid exposing incognito-status to the application. However, we avoid propagating the actual CaptureHandle between the capturing app and the captured app.

# Upcoming Extensions
* The current Capture Handle only applies to tab-capture.
* It is possible to extending the concept to browser windows, where the captured window's active tab determines the capture-handle.
* Extensions to native applications are possible, but are trickier, likely requiring OS-level support.

# Goals and Non-Goals

Communication often presupposes that both sides can recognize each other. Previously, a capturing application had no way to ergonomically and reliably detect which application was captured.  It is this gap that we seek to address - **identification**. How communication proceeds is left entirely in the hands of the capturing+captured applications.

Two noteworthy possible ways for communication to proceed are [BroadcastChannel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel/BroadcastChannel) or a shared cloud infrastructure. Virtually all indirect communication methods presuppose that some **session ID** is used. For example, when using a BroadcastChannel, it would likely be useful to address messages to the specific tab being captured, and not to all tabs of that origin (consider multiple tabs with different presentations, only one of which is being shared by the user).

# Consideration of Alternative Approaches

## Rejected Alternative #1: MessageChannel on MediaStreamTrack
### Idea
Add a [MessageChannel](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel) to the MediaStreamTrack and allow the two applications to communicate directly.

### Drawbacks
* This proposal lacks a convenient way for the capturing application to identify the captured application, and determine the protocol needed for communication. (I.e. what messages may be sent, that would be understood.)
* If this approach is extended to include controlled exposure of the captured application's origin, then the approach becomes a more complex variant of Capture Handle, that also requires at least one RTT between the apps before anything useful can be done by the capturing application. (This limits the usefulness of Capture Handle for the upcoming Conditional Focus feature - link pending.)
* In either case, the capturing application is forced to alert the captured application to the presence of a display-capture session.
* Difficulties arise when considering that capture sessions can stop, restart, and that multiple applications could be capturing the same tab - but that the browser does not wish to alert the captured application to any of these developments. (It's OK if the capturing application does that, though.)

## Rejected Alternative #2: On-Rails Approach
### Idea
Define a closed set of messages that can be sent from the capturer to the captured, e.g. by extending `MediaStreamTrack ` with a `jumpToSlide(num)` method.

### Drawbacks
* This is a very partial solution, addressing only the subset of use-case #1 where the captured application has slides. Even then, it is unlikely that our imagination is going to be enough to think of all required actions, express them all in the form of simple actions with predetermined parameters. 
* The lack of an identification mechanism, and therefore of an authentication mechanism, means that collaboration between the capturing and captured applications would be limited to a set of simple, unprivileged actions which the captured application would be willing to accept from an arbitrary capturing application.
