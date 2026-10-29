# speed-limits-models

The neural networks that read Swiss speed-limit signs from the road camera of a comma, for the sunnypilot branch
[`speed-limits`](https://github.com/cristianku/sunnypilot/tree/speed-limits). They are trained and measured in
[`sunnypilot.speedlimits`](https://github.com/cristianku/sunnypilot.speedlimits).

sunnypilot takes this repository as a git submodule at `openpilot/sunnypilot/speed_limits/models`, the way it takes
its lateral models from `neural_network_data`. The device compiles the ONNX files for its GPU with tinygrad the first
time it needs them (offroad, a minute or two), not the build: a network that fails to compile leaves only the speed
limit reading out. A new model is a commit here and a move of the submodule pointer in the branch.

## What is here

- `speed_limits.json`: the networks in use and their version (format 2).
- `signnet.onnx`: SignNet, the finder. Input `images` `(1, 3, 288, 896)`, the top 288 rows of the road frame resized
  to 896 wide, RGB 0..1. Output `out` `(1, 5, 36, 112)`: a heat map at stride 8 and the size of the sign there.
- `classifier.onnx`: the classifier. Input `images` `(1, 3, 160, 160)`, the crop of a box found by SignNet,
  ImageNet-normalised. Output `logits` `(1, 41)`; the classes are in its contract (`output.classes`).
- `road-type.onnx` (when `speed_limits.json` has `road_type`): the road-type network, one step a second with its
  memory. Inputs `image` `(1, 3, 144, 256)`, the whole road frame resized, RGB 0..1; `speed`, `sign` (seconds since
  a motorway or expressway sign), `motion` (the car's turn, steering and blinkers) and the memories `hidden`,
  `junction_hidden` `(1, 128)`. Output `probabilities` `(1, 6)` over town, rural, expressway, motorway, ramp, parking
  (its contract, `output.classes`) and the memories for the next step. The camera and the car only: never the map.
- `*.onnx.contract.json`: each network's inputs, outputs, preprocessing and the hash of its ONNX.

## How sunnypilot uses them (format 2)

`speedvisiond` reads 5 frames a second while onroad, when Show Vision Speed Limit is on (Cruise › Speed Limits) or
Speed Limit Source is Map Data + Vision. On each frame:

1. SignNet finds up to 5 signs in the top of the frame (local maxima of the heat map, score 0.3 or more);
2. the classifier reads each: a speed sign (maximum speed, variable display, zone, road marking) gives its number,
   the end of a limit gives 0, anything else nothing; readings under 70 % confidence are dropped;
3. a value becomes the limit when two frames within 1.5 s read it; it stays until another value is confirmed
   (two values at once: the one seen most, then the lower).

Once a second the road-type network sees the whole frame; its answer counts after 3 equal steps. Where no sign gives
the limit, the scene does:

- a car park (and the car under 25 km/h for 5 s) → 15 km/h, a sign there only when lower;
- out of the car park, or at the start of a drive with nothing read yet → the general limit of the road it sees
  (in Switzerland a town 50, rural 80, expressway 100, motorway 120);
- the limit of the scene follows another kind of road after 10 s of it; a sign read always wins over the scene.

It publishes `speedVisionSP` (`speedLimit` in m/s, 0 when there is no reading yet; `roadType`, what the road-type
network sees). The preprocessing is the
training's, bit for bit: ffmpeg's YUV to RGB and Pillow's bilinear resize.

The reading is shown on the screen and is advisory input to sunnypilot's Speed Limit Assist in Speed Limit Source ›
Map Data + Vision; nothing here controls the car.

## Where it is going: one graph

The goal is one ONNX with the whole chain, the memory included (plain tensor operations, no weights):

| name | dir | shape | meaning |
|---|---|---|---|
| `frame` | in | road camera frame | |
| `state_in` | in | `[16]` float | `state_out` of the previous frame; zeros at the start of a drive |
| `general_limits` | in | `[4]` float | town, rural, expressway, motorway limits of the country the car is in |
| `limit_kph` | out | `[1]` float | the limit in force, or 0 when the model is not sure |
| `state_out` | out | `[16]` float | passed back as `state_in` on the next frame |

0 means "not sure": sunnypilot's Speed Limit Source then takes the map, exactly as when a car reads no sign. The order,
strongest first:

1. a speed sign read now → that limit (inside a car park see 2);
2. a car park recognised → 15 km/h, a sign read there only when lower; the limit remembered from the road before is
   dropped (a car park has no legal limit, but the map often has none there either: 0 could let the car speed up);
3. the limit remembered from the last sign → that limit;
4. the general limit after the sign that starts it (in Switzerland "50 generell", the motorway or expressway start),
   no speed sign since, road type sure → that general limit;
5. no sign read on this road (the start of a drive, out of a car park), road type sure → the general limit of the
   road the scene shows (a town 50);
6. anything else → 0.
