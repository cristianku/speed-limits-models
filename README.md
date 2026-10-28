# speed-limits-models

The neural networks that read Swiss speed-limit signs from the road camera of a comma, for the sunnypilot branch
[`speed-limits`](https://github.com/cristianku/sunnypilot/tree/speed-limits). They are trained and measured in
[`sunnypilot.speedlimits`](https://github.com/cristianku/sunnypilot.speedlimits).

sunnypilot takes this repository as a git submodule at `openpilot/sunnypilot/speed_limits/models`, the way it takes
its lateral models from `neural_network_data`. The build compiles the ONNX for tinygrad on the device, as it does for
the driving model. A new model is a commit here and a move of the submodule pointer in the branch.

## What is here

- `speed_limits.json`: the model in use (`model`: the ONNX file, null while there is none yet) and its version.
- `*.onnx`: one file per version, the whole chain in one graph: cutter (with the road-type head), sign classifier,
  and the memory of the signs read (plain tensor operations, no weights).

## The contract with sunnypilot

Every frame, `speedvisiond` runs the model and publishes `speedVisionSP`:

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
5. anything else → 0.

The limit is advisory input to sunnypilot's Speed Limit Assist, chosen in Speed Limit Source › Map Data + Vision;
nothing here controls the car.
