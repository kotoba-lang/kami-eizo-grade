# kami-eizo-grade

Portable `.cljc` color-grading data model + math — the `eizo` (映像,
video) domain analog of what DaVinci Resolve's Color page computes
internally. Part of the `kami-eizo` family defined by ADR-2607121400.

- `kami.eizo.grade.cdl` — ASC CDL (Slope/Offset/Power) primary color
  correction (`out = ((in * slope) + offset) ^ power` per channel, the
  same math underlying every "Lift/Gamma/Gain" 3-wheel UI), plus
  luma-preserving saturation (Rec. 709 luma coefficients).
- `kami.eizo.grade.curve` — per-channel tone curves via monotone cubic
  Hermite interpolation (Fritsch-Carlson tangent limiting): monotonic
  control points are guaranteed to produce a monotonic curve between
  them (no overshoot), unlike plain Catmull-Rom.
- `kami.eizo.grade.lut` — Adobe/Iridas `.cube` 3D LUT parser + trilinear
  sampling.
- `kami.eizo.grade.scope` — waveform (luma histogram) and vectorscope
  (Rec. 709 YCbCr chroma-plane) *data* computation over a flat pixel
  sequence.
- `kami.eizo.grade` — composes CDL -> curves -> LUT into one pipeline
  (`apply-grade`) and wraps it as the opaque effect-instance shape
  `kami-eizo-timeline` expects on a clip/track's effect stack
  (`effect-instance`).
- `kami.eizo.grade.mathutil` — portable `:clj`/`:cljs` scalar math
  (`Math/pow` et al. under different symbols per platform).

## v0 scope — what this is NOT

- No image/video decode or file I/O — a "frame" is just a flat sequence
  of `[r g b]` pixel triples you provide.
- No viewport/GPU rendering of the waveform/vectorscope — this computes
  the underlying data (histogram / chroma-vector points) only; drawing
  it is an app/GPU-layer concern (existing `webgpu`/`webgl` per
  ADR-2607121400 §2.1, no second renderer here).
- Curve interpolation is monotone cubic Hermite, not a full spline
  editor (no per-segment tension/bias controls).
- `.cube` parsing assumes the common `DOMAIN_MIN 0 0 0` / `DOMAIN_MAX 1
  1 1` case; non-default domain remapping is not implemented.

## Usage

```clojure
(require '[kami.eizo.grade :as g]
         '[kami.eizo.grade.cdl :as cdl]
         '[kami.eizo.grade.curve :as curve]
         '[kami.eizo.grade.lut :as lut])

(def node (g/grade {:cdl-node (cdl/cdl {:slope [1.1 1.0 0.95] :offset [0.02 0.0 0.0] :power [1.0 1.0 1.05]})
                     :curves {:r (curve/curve [[0.0 0.0] [0.5 0.55] [1.0 1.0]])}}))

(g/apply-grade node [0.4 0.5 0.6])
;; => [r g b] with primary + curve applied

(g/effect-instance node)
;; => {:type :grade :params node :enabled? true}   ; drop into a
;;    kami-eizo-timeline clip/track effect stack
```

## Test

```bash
clojure -M:test
clojure -M:lint
```

## License

Apache-2.0
