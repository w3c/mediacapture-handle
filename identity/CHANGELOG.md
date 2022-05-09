# API Changelog

## Chrome m92
API introduced and exposed as an origin trial.

## Chrome m93
* Capture handle previously exposed `track.getSettings().captureHandle`; now as `track.getCaptureHandle()`.
* Events previously contained the capture handle as `event.captureHandle`, now as `event.captureHandle()`.

## Chrome m102
* `CaptureHandleChangeEvent` has been replaced by a simple `Event`. <http://crbug.com/1322174>
