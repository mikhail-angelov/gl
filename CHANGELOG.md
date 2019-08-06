# @sveltejs/gl changelog

## 0.0.17

* Default to crossOrigin images

## 0.0.16

* Allow meshes to receive frag, vert, blend properties

## 0.0.15

* Declare extensions first, due to change in Chrome 75

## 0.0.14

* Allow `pixelRatio` to be specified

## 0.0.13

* Fetch image data in `cors` mode

## 0.0.12

* Only update when canvas is visible

## 0.0.10-11

* Fix resize glitches

## 0.0.9

* Support primitives other than lines

## 0.0.8

* Fix blending formula

## 0.0.7

* Default to `highp` floats
* Work around some shader bug I don't understand
* Depth test by default

## 0.0.6

* Fix serialization of minified code sent to workers

## 0.0.5

* Add `<Target>` component
* Add `user-select: none` on overlays
* Load and decode images off the main thread
* Allow depth testing to be disabled per-material

## 0.0.4

* Bump mapping
* New `Sphere` implementation

## 0.0.3

* Don't reinitialise element buffers unnecessarily
* Enable `minDistance` and `maxDistance` on OrbitControls

## 0.0.2

* Throttle OrbitControls events
* Add touch support to OrbitControls

## 0.0.1

* First experimental release