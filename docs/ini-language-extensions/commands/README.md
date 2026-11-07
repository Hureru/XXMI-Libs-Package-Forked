# Commands

## `store`

Reads a 32-bit floating-point value from a GPU resource and stores it in an INI runtime variable or a [pool variable](../pools/variables.md).

```ini
store = $out, ResourceFoo, $offset
```

> ⚠️ **`store` is VERY expensive** unless the value comes from a [cached buffer](#cached-reads). A readback stalls the CPU until the GPU has finished all queued work. Used on every draw call, or even every frame, it **will kill the frame rate**. Gate it so it runs rarely, see [GPU Readback](../examples/gpu-readback.md).

The command takes three arguments:

1. **Output variable** — the INI variable or pool variable that receives the value, such as `$out` or `$PoolFoo[$index]`.
2. **Resource target** — the pipeline slot or custom resource to read from.
3. **Offset expression** — a dynamically evaluated expression specifying which 32-bit value to read.

For example:

```ini
store = $out, ResourceFoo, 4
```

The offset is interpreted as a **32-bit value index**, so `4` reads the fifth 32-bit value from the resource.

### GPU Readback

`store` performs **GPU → CPU readback**. The requested data is copied from the GPU resource into a CPU-readable staging buffer and then read by the CPU.

Only the required data is read back whenever possible. For regular buffers, only the requested 4-byte value is copied. For structured buffers, the smallest range containing the required complete structures is copied.

Readback staging buffers are reused across `store` invocations rather than recreated for every call.

Despite these optimizations, GPU → CPU readback remains **very expensive**: the CPU has to wait for the GPU to finish everything it was given before the value can be read, which serializes the two and can cut the frame rate by a large factor when done frequently. `store` should therefore only be used when the value is actually needed by the CPU-side INI runtime, there is no other way around it, and the call is guarded so it does not execute on every draw or frame.

### Cached Reads

Buffer types listed in `cache_resource_data` in the `[Rendering]` section are kept in RAM as the game writes them through `Map` or `UpdateSubresource`. `store` reads such a value straight from that copy, with no staging buffer and no stall:

```ini
[Rendering]
cache_resource_data = constant_buffer
```

The cache only holds what the CPU wrote. A buffer the GPU produced, such as the output of a custom compute shader, still takes the readback path above, and so does any buffer of a type that is not cached.

See [GPU Readback](../examples/gpu-readback.md) for practical usage patterns.

## `wrap_draws`

Direct3D 12 only. For the rest of the draw of the game it runs in, every draw made there (the game's own, unless `handling = skip` said otherwise, and every `draw`/`drawindexed` of a mod) is made by running the given command list in its place. In that list `draw = from_caller` makes the draw it stands in for, with its own count, start and base vertex; whatever else it draws is drawn as it comes and is not wrapped again.

```ini
[TextureOverrideBody]
hash = 12345678
match_first_index = 0
wrap_draws = CommandListTwoPasses

[CommandListTwoPasses]
run = CustomShaderDepthPrepass
run = CustomShaderEqualDepth
```

`wrap_draws = null` makes the draws that follow as they come again. The setting ends with the draw of the game it was made in. It is what a [ShaderRegex] section that rewrites the game's shader does under D3D11, where the shaders of a D3D12 game cannot be rewritten: making several passes of every draw of a mesh, whoever draws it. It is ignored for D3D11 and Vulkan games and in dispatches, with a note in the dump of `dx12_dump_hashes = 1`, whose report also counts the wrapped draws.

### Fixed-function state of `[CustomShader]` sections

A draw a mod makes in a draw of a D3D12 game is made with the game's pipeline state made again with the mod's shaders. The blend, depth and stencil and rasterizer state a `[CustomShader]` section sets go into that state for the draws of its command lists: in full, as they would replace what is set on a D3D11 context, or with `blend_state_merge = 1`, `depth_stencil_state_merge = 1` and `rasterizer_state_merge = 1` only what the section names, the rest staying as the game has it. `stencil_ref` and `blend_factor[N]` are set on the command list for those draws.

```ini
[CustomShaderEqualDepth]
; the game's shaders, drawn only where the depth is what an earlier pass left
depth_func = equal
depth_write_mask = zero
depth_stencil_state_merge = 1
draw = from_caller
```

A `mask[N] = 0x...` alone changes only which channels are written: a target the game does not blend stays unblended, and one it blends keeps its blending. Where D3D12 refuses the pipeline, the draw is made with the game's state and shaders, and the dump says why.
