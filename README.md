# shapes — Procedural 3D Geometry & Mesh Engine for Edva

`shapes` is a procedural 3D geometry and mesh-generation library written in [Edva](https://gitea.gde.to/ike/dva). It provides builders for the **Five Platonic Solids** (Tetrahedron, Cube, Octahedron, Dodecahedron, Icosahedron), point-based polyhedra, curved parametric surfaces (Sphere, Cylinder, Cone, Torus), and **3D parallel-transported polar cross-section filaments** with decoupled RGBA color providers.

---

## Features

- **Decoupled RGBA Coloring**: Builders never hardcode colors. Polyhedra and curved surfaces accept face/sector color providers (`Int => Color`), and filaments accept polar-axial color providers (`(angle: Float, dist: Float => Color)`). Includes combinators for `uniform`, `glass` (translucent alpha), `gradient` (linear & Oklab), `rainbow`, `warm`, `cool`, and canonical Platonic palettes.
- **Exact Platonic Solid Geometry**: Uses golden ratio $\phi$ (`const::phi`), inverse golden ratio $\phi^{-1}$ (`const::inv_phi`), and root constants directly from `std/const`.
- **Automatic Surface Normals**: Flat face normals via normalized cross products (`geom::normal3`) and analytical smooth surface derivatives (`curve::polar_normal`) for curved and polar cross-section shapes.
- **3D Curve Framing & Parallel Transport (`shapes/curve`)**: Supports both discrete polyline point sequences `geom::Vec3{}` and continuous parametric curves `(Float => geom::Vec3)` via the `FilamentSpine` choice type, propagating twist-free Bishop frames (`initial_frame`, `transport_frame`) along any 3D curve.
- **Direct 12-Float RGBA+UV GPU Vertex Layout (`VERTEX_STRIDE = 48` bytes)**: Writes interleaved `(px, py, pz, cr, cg, cb, ca, nx, ny, nz, u, v)` `f32` vertices directly into target memory buffers, paired with `shapes::Mesh` (`(start: Int, count: Int)`) slices for clean draw-call submission.
- **Procedural 2D RGBA8 & Reflective Environment Textures (`shapes/texture`)**: Rasterizes any `(u: Float, v: Float => Color)` sampler (`checkerboard`, `grid`, `stripes_u`, `stripes_v`, `brick`, `dots`, `marble`, `wood`, `reflective`, `chrome`) into an RGBA8 `Texture` buffer ready for `gl.texImage2D`.
- **Zero WebAssembly Allocations**: Safe for high-frequency 60 FPS WebGL/WebGPU render loops without garbage collection or heap churn.

---

## Installation

Add `shapes` (and `mat` if needed) to your `edva.ccl`:

```ccl
dependencies =
  shapes =
    path = ../shapes
```

---

## Module Overview

| Module | Core Types / Symbols | Description |
| :--- | :--- | :--- |
| [`src/vertex.dva`](src/vertex.dva) | `Vertex: #packed (12 * f32)`, `VERTEX_STRIDE = 48`, `Mesh: (start, count: Int)`, `mesh(start, end_idx)` | 48-byte RGBA+UV interleaved vertex layout (`pos3, rgba4, norm3, uv2`), low-level `write_vertex` emitters, and `Mesh` slice tracking. |
| [`src/palette.dva`](src/palette.dva) | `tetrahedron`, `cube`, `octahedron`, `dodecahedron`, `icosahedron`, `curved` | Canonical high-contrast face palettes built on `std/color` (`#use @shapes/palette`). |
| [`src/texture.dva`](src/texture.dva) | `Texture`, `generate`, `checkerboard`, `grid`, `stripes_u`, `stripes_v`, `brick`, `dots`, `marble`, `wood`, `reflective`, `chrome` | Procedural 2D RGBA8 texture generator, UV surface samplers, and spherical reflection maps (`#use @shapes/texture`). |
| [`src/polyhedra.dva`](src/polyhedra.dva) | `write_tetrahedron`, `write_cube`, `write_octahedron`, `write_dodecahedron`, `write_icosahedron`, `write_plane`, `write_star_base` | Platonic solids, arbitrary point-based polyhedra (`write_tetrahedron_4pt`, `write_octahedron_6pt`, `write_hexahedron_8pt`), and planar helpers. |
| [`src/curve.dva`](src/curve.dva) | `FilamentSpine`, `spine_points`, `spine_curve`, `resolve_spine`, `curve_tangent`, `initial_frame`, `transport_frame`, `polar_normal` | 3D parametric curve sampling, central-difference tangents, parallel-transported Bishop normal/binormal frames, and analytical polar cross-section normals. |
| [`src/curved.dva`](src/curved.dva) | `write_sphere`, `write_cylinder`, `write_cone`, `write_torus`, `write_filament` | Smoothly shaded parametric surfaces and generalized polar cross-section tubes/filaments. |
| [`src/shapes.dva`](src/shapes.dva) | Umbrella geometry & mesh API | Exports vertex/mesh types, curve helpers, and geometry builders. |

---

## Quick Usage

### 1. Platonic Solids, Glass Colors & `shapes::Mesh` Slices

```dva
#use color
#use @shapes
#use @shapes/palette

// Allocate vertex staging buffer
buf = shapes::vertex_buffer(4096)
k' = 0

// 1. Canonical multi-color Platonic solid
s0 = k'
k' = shapes::write_dodecahedron(buf, k', 1.2, palette::dodecahedron)
dodec_mesh = shapes::mesh(s0, k') // Mesh((start: 0, count: 108))

// 2. Translucent tinted glass icosahedron (alpha = 0.36)
s1 = k'
ruby_glass = color::glass(color::uniform(color::rgb(0.95, 0.22, 0.42)), 0.36)
k' = shapes::write_icosahedron(buf, k', 1.0, ruby_glass)
ico_mesh = shapes::mesh(s1, k') // Mesh((start: 108, count: 60))
```

### 2. Parametric 3D Filament with Polar Cross-Section & Axial Color Gradient

`shapes::write_filament` sweeps a polar cross-section `radius_fn(angle, dist)` and surface color `color_fn(angle, dist)` (`angle` $\in [0, 2\pi]$, `dist` $\in [0, 1]$) along a `FilamentSpine`—which can be either a parametric curve function `Float => geom::Vec3` (`shapes::spine_curve`) or a discrete point array `geom::Vec3{}` (`shapes::spine_points`):

```dva
#use const
#use math
#use "math/geom"
#use color
#use @shapes

// 1. Define a 3D parametric helix spine: dist in [0.0, 1.0] => Vec3
helix_spine: Float => geom::Vec3
helix_spine = dist =>
   ang = dist * 2.0 * const::tau
   geom::vec3((dist - 0.5) * 3.0, math::cos(ang) * 0.4, math::sin(ang) * 0.4)

// 2. Define a 5-lobed star cross-section that twists along the spine
star_radius: Float, Float => Float
star_radius = angle, dist =>
   0.25 * (1.0 + 0.30 * math::cos(5.0 * angle - dist * 3.0 * const::tau))

// 3. Smooth glass gradient from sapphire blue (dist=0) to emerald green (dist=1)
sapphire_to_emerald: Float, Float => color::Color
sapphire_to_emerald = angle, dist =>
   axial = color::gradient(
      color::rgba(0.12, 0.52, 0.96, 0.46),
      color::rgba(0.04, 0.82, 0.45, 0.52)
   )
   axial(dist)

// 4. Emit 320 axial slices × 180 radial segments
k_end = shapes::write_filament(
   buf, 0,
   shapes::spine_curve(helix_spine),
   320, 180,
   star_radius,
   sapphire_to_emerald
)
filament_mesh = shapes::mesh(0, k_end)
```

---

## Running Tests

```bash
cd ~/Z/edva-ware/shapes
edva test
```
