---
name: slang-coding
description: Write, review, and refactor .slang shader code using idiomatic Slang features instead of treating Slang as renamed HLSL. Enforce readable formatting, maintainable shader architecture, deliberate parameter binding, cross-target correctness, and host ABI compatibility. Use when creating, modifying, porting, or reviewing Slang shaders.
---

# Slang coding

Use this skill for **Slang shader language** (`.slang`) files. Favor modern Slang syntax and abstractions while respecting GPU execution, target capabilities, and the application's pipeline interface. Do not treat a `.slang` file as an `.hlsl` file with a different extension.

## Non-negotiable instructions

1. **Read the context before editing.** Identify the Slang compiler version, target backends (SPIR-V/Vulkan, DXIL/D3D12, Metal, WGSL/WebGPU, CUDA, etc.), shader stages, imported modules, existing formatting conventions, and build commands.
2. **Find the host-side contract.** Look for pipeline entry-point names, descriptor-set/binding expectations, buffer layouts, vertex attributes, push constants, specialization choices, and reflection-generated metadata. Never silently change these.
3. **Prefer native Slang for new code.** Use `let`/`var`, `module`/`import`, visibility control, `ParameterBlock<T>`, generics, interfaces, and extensions **when each solves a real problem**. Do not introduce abstractions merely to show off a language feature.
4. **Make shader intent explicit.** Name coordinate spaces, units, shape modes, transforms, colors, and thresholds. Avoid packed flags and magic numbers unless preserving an existing ABI.
5. **Format every touched region.** No one-line control-flow bodies, unrelated declarations crammed together, or dense expressions that conceal behavior.
6. **Verify changes.** Compile with the project's actual Slang version and target settings, inspect warnings, and validate generated binaries or runtime results where tooling exists. Do not claim compilation or rendering succeeded if it was not run.
7. **Explain intentional interface changes.** If a better Slang design requires different buffers, descriptors, vertex formats, or CPU-side structures, present it as a migration and list required host changes. Otherwise preserve compatibility.

## Formatting contract

Follow the repository's established style if it is consistent. For new code, use this default:

| Element                | Rule                                                                                                                                |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Indentation            | 4 spaces; never tabs                                                                                                                |
| Braces                 | Allman style: opening `{` on its own line for types, functions, and control flow                                                    |
| Braces on conditionals | Always use braces, including single-statement `if`, `else`, loops, and `switch` cases with local declarations                       |
| Line width             | Aim for 100 columns; use up to 120 when a break would harm clarity                                                                  |
| Spacing                | One space around binary operators; after commas; before control-flow parentheses; no padding just inside `()` or `[]`               |
| Blank lines            | One between top-level declarations and logical sections; separate stages and helpers clearly                                        |
| Semicolons             | Required after variable statements and structure declarations; not after function bodies                                            |
| Comments               | Explain shader intent, coordinate conventions, ABI assumptions, performance tradeoffs, and non-obvious math—not line-by-line syntax |
| File organization      | Module/imports, shared constants/enums, data types, parameters/resources, helpers, entry points                                     |
| File names             | Use meaningful `.slang` names consistent with the project; reusable library modules get their own files                             |

### Naming

- **Types, interfaces, enums:** `PascalCase` (`VertexOutput`, `ShapeKind`, `ILightingModel`).
- **Functions, locals, fields, shader parameters:** `camelCase` (`fragmentMain`, `localPosition`, `drawParameters`). Preserve existing host-reflected names when changing them would break integration.
- **Constants:** `kPascalCase` (`kUnitRadius`, `kQuadVertices`) or the repository's consistent constant scheme.
- **Enum cases:** `PascalCase` (`ShapeKind.Triangle`).
- **Shader stages:** name functions after their stages and tasks (`vertexMain`, `fragmentMain`, `computeMain`) unless the pipeline already expects specific symbols.
- **Semantic types:** use descriptive words (`clipPosition`, `localUv`, `worldNormal`, `viewDirection`) instead of `p`, `v`, `tmp`, or ambiguous `transform` components.

### Good local declarations

```slang
let radiusSquared = dot(position, position);
var opacity : float = 1.0;
opacity *= coverage;
```

Use `let` by default for local values that are not reassigned; use `var` when mutation is needed. Explicit type annotations (`let x : float = ...`) are useful for API boundaries, precision-sensitive calculations, and avoiding surprising inference. Standard Slang/HLSL vector types such as `float2`, `float3`, and `float4` are **correct Slang**; do not invent replacement types.

## Slang-first language design

### Variables, types, and control flow

- Prefer `let`/`var` for local variables rather than declaring all locals in traditional HLSL style. A conventional type-first declaration is still valid where it is clearer or required by existing style.
- Prefer named `enum` cases over numeric mode values when you control both shader and host data. Check the target/compiler behavior before using special enum underlying types; an integer field with an explicit conversion can be safer at a CPU/GPU boundary.
- Keep functions small and single-purpose. Factor repeated or conceptually distinct math into helpers such as `isInsideShape`, `transformPosition`, or `evaluateLighting`.
- Use `switch` for a clear set of discrete modes and avoid accidental fallthrough. Prefer runtime branching for genuinely dynamic decisions and compile-time specialization for deliberately fixed variants.
- Avoid implicit, unclear casts; document units and coordinate spaces. Keep floating-point constants and precision deliberate (`0.5`, `1.0`, etc.).
- Use Slang's generics, interfaces, and extensions to express reusable algorithms or interchangeable implementations—not to add ceremony to a one-off pixel shader.

### Modules instead of textual includes

Use Slang modules for reusable shader libraries:

```slang
module ShapeMath;

public bool isInsideUnitCircle(float2 position)
{
    return dot(position, position) <= 1.0;
}
```

```slang
import ShapeMath;

```

- Prefer `import SomeModule;` over `#include` for reusable code; modules support access control and avoid preprocessor-state leakage.
- Declare `module ModuleName;` in a module's primary source file and mark intended external APIs `public`; keep implementation details non-public.
- Use `implementing ModuleName;` and `__include` only when intentionally splitting **one** module across multiple files; this is different from C preprocessor `#include`.
- Keep `#define` for conditional compilation or required integration switches, not for type specialization, function overloading, or string-pasted variants that Slang generics/interfaces can express.
- Match the project's module lookup paths, names, and compiler-version support.

### Shader parameters and resources

| Need                                                                           | Prefer                          | Why / caution                                                                                         |
| ------------------------------------------------------------------------------ | ------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Cohesive group of uniforms **and/or resources**, managed as one host-side unit | `ParameterBlock<T>`             | Naturally groups a binding set/table; can change descriptor-set assignments relative to legacy code   |
| Plain uniform-buffer data with an existing descriptor/buffer contract          | `ConstantBuffer<T>`             | Explicitly represents constant-buffer-backed data; does **not** mean “one independent descriptor set” |
| Reusable data layout                                                           | Named `struct`                  | Documents fields and can support reflection-driven host code                                          |
| Fixed Vulkan bindings imposed by the engine                                    | `[[vk::binding(binding, set)]]` | Keep only when required by the host layout or an external API contract                                |
| Fixed D3D register spaces imposed by the engine                                | `: register(..., space...)`     | Keep only when required for DX/HLSL compatibility                                                     |
| Ordinary loose global uniforms                                                 | Use deliberately                | Reflect and understand the compiler's automatic grouping; do not assume they are push constants       |

**Important distinctions:**

- Slang can assign bindings deterministically; prefer automatic binding **plus reflection** for new integration code. Never guess host-visible binding numbers.
- `ParameterBlock<T>` is not a cosmetic replacement for `ConstantBuffer<T>`: on Vulkan it generally introduces a separate descriptor set for the group's resources; ordinary data in the block can become a uniform-buffer binding. Confirm the actual generated layout using reflection.
- Do not apply both Vulkan and D3D annotations by habit. If targeting both APIs with fixed bindings, maintain each contract intentionally.
- Preserve buffer field order, sizes, padding, matrix major order, scalar types, descriptor sets, binding indices, and shader stage visibility during a compatibility-only refactor.
- A `float4` containing `xy = translation`, `zw = scale` may be required by an existing ABI; document the packed layout rather than silently splitting it into `float2` fields. When allowed to redesign, use named fields.
- Keep unrelated parameters in distinct fields. For Vulkan push constants, follow the platform-specific Slang syntax, size limits, and host layout rather than assuming all small uniforms become push constants.

### Entry points and stage interfaces

- Use `[shader("vertex")]`, `[shader("fragment")]`, `[shader("compute")]`, etc., on the actual entry points; these attributes are part of modern Slang practice, not obsolete HLSL baggage.
- Keep required semantics such as `SV_Position`, `SV_Target0`, `SV_VertexID`, and `TEXCOORD0`. Slang needs them for stage inputs/outputs and cross-target mapping.
- Use named structs for nontrivial stage I/O, and ensure matching varyings have compatible ordering, types, and semantics across backends.
- Initialize all output fields before returning. Do not rely on undefined values, uninitialized variables, or implicit stage conversions.
- Use explicit coordinate conventions (e.g., `uv` in `[-1, 1]` versus texture UV in `[0, 1]`) and do not call a position-like value a UV unless documented.

## Performance and correctness

- **Correctness first:** preserve raster coverage, interpolation, derivatives, depth behavior, blend-relevant alpha, clipping, and alpha/discard behavior unless the task calls for a visual change.
- Avoid gratuitous per-fragment work. Compare squared distances with `dot(v, v)` when mathematically appropriate; preserve exact threshold semantics when a bit-for-bit-compatible refactor is required.
- Do not assume `discard` is free or equivalent to alpha blending; it can affect depth/early-Z behavior and derivative-dependent operations.
- Do not replace straightforward branches with `step`, `lerp`, dynamic indexing, or branchless tricks without target-specific evidence that the change helps.
- Avoid divergent control flow around gradient operations or barriers. Make compute barriers uniform across participating threads.
- Choose resource types and read/write access deliberately (`Texture2D<T>`, `SamplerState`, `StructuredBuffer<T>`, `RWStructuredBuffer<T>`, etc.). Never assume resource indices or array bounds are valid.
- Avoid duplicated literals: define named constants for geometric thresholds, workgroup sizes, and numerical tolerances when they communicate a stable design decision.
- Use `float`/`half`/integer widths based on precision needs and supported target behavior; do not narrow types as a blind optimization.
- Prefer straightforward, measurable shader code over speculative micro-optimizations or abstractions that bloat generated code.

## Review checklist for every `.slang` change

- [ ] Is this idiomatic Slang, not reflexively HLSL-style code? Are `let`/`var` used sensibly?
- [ ] Are the braces, spacing, blank lines, indentation, and line wrapping consistent?
- [ ] Are variable names and data layouts descriptive? Are magic values named where appropriate?
- [ ] Could `module`/`import`, a helper, an enum, a generic, or an interface remove genuine duplication?
- [ ] Are `ParameterBlock<T>` and `ConstantBuffer<T>` chosen for their **different semantics**?
- [ ] Are manual bindings present only for documented host/API requirements?
- [ ] Are binding reflection, entry-point names, shader stages, semantics, and buffer layout compatible?
- [ ] Are output values initialized and coordinate-space conventions explicit?
- [ ] Are branches, `discard`, numerical thresholds, derivatives, and memory operations correct?
- [ ] Has the shader been compiled for each affected target, with warnings handled?
- [ ] Were runtime/render tests performed where possible, or explicitly identified as not run?

### Compile/validation workflow

A single-entry SPIR-V check, when these tools are available:

```sh
slangc shader.slang -target spirv -entry vertexMain -o /tmp/shader.spv
spirv-val /tmp/shader.spv
```

Use the repository's real compile flags, profiles, capabilities, module search paths (`-I`), entry points, and target versions. When multiple backends are supported, compile **each** relevant backend and compare reflected layouts. A shader that compiles is not necessarily visually correct: verify draw calls, buffer data, descriptor bindings, topology, and pixel output in the renderer. If `slangc` or `spirv-val` is unavailable, state this accurately rather than inventing test results.

## References (primary sources)

- [Slang user guide](https://docs.shader-slang.org/en/latest/external/slang/docs/user-guide/)
- [Language declarations (`let`, `var`, enums)](https://docs.shader-slang.org/en/latest/external/slang/docs/language-reference/declarations.html)
- [Modules and access control](https://docs.shader-slang.org/en/latest/external/slang/docs/user-guide/04-modules-and-access-control.html)
- [Parameter blocks](https://docs.shader-slang.org/en/latest/parameter-blocks.html)
- [Shader cursors, reflection, and resource layout](https://docs.shader-slang.org/en/latest/shader-cursors.html)
- [SPIR-V-specific behavior](https://docs.shader-slang.org/en/latest/external/slang/docs/user-guide/a2-01-spirv-target-specific.html)
- [Command-line options](https://docs.shader-slang.org/en/latest/external/slang/docs/command-line-slangc-reference.html)

**Agent output expectation:** Deliver formatted Slang code, a concise explanation of intentional language/architecture choices, a list of any CPU/GPU interface changes, and the actual compilation/validation results (or an explicit note that they were not run).
