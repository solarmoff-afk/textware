# textware

A high-performance, cross-platform text rendering engine cosmic-text based for `wgpu`

`textware` wraps `cosmic-text` and `swash` to provide a simple, GPU-accelerated text rendering pipeline. It handles complex text shaping, font management, and automatic texture atlas packing, generating ready-to-draw vertex buffers

## Features

*   **Zero-Overhead Font Management**: Load font data once, use it everywhere with lightweight IDs.
*   **Automatic Atlas Packing**: Glyphs are rasterized and packed into a dynamic GPU texture atlas.
*   **Complex Layouts**: Supports multi-line text, wrapping, alignment, and dynamic sizing.
*   **Mobile Ready**: First-class support for **iOS** and **Android** (via Native AssetManager).
*   **WGPU v24**: Built for the latest WebGPU standards.

## Platform Support

*   **Windows**
*   **Linux**
*   **macOS**
*   **iOS**
*   **Android**
*   **FreeBSD / OpenBSD / NetBSD**

## Usage

### 1. Initialization

Initialize `TextWare` with your wgpu device and queue.

```rust
use textware::TextWare;

// Desktop & iOS
let mut textware = TextWare::new(&device, &queue);

// Android (requires ndk::asset::AssetManager)
// let mut textware = TextWare::new(&device, &queue, asset_manager);
```

### 2. Loading Fonts

Load fonts from the filesystem (or assets on Android) or raw bytes.

```rust
// Load from file
let roboto_id = textware.load_font_file("fonts/Roboto-Regular.ttf").unwrap();

// Load from bytes
let font_bytes = include_bytes!("my_font.ttf");
let custom_id = textware.load_font_bytes(font_bytes, "MyFont").unwrap();
```

### 3. Creating Text

Create text objects. You define the font size and line height here.

```rust
// create_text(content, font_id, font_size, line_height)
let mut title = textware.create_text("Hello World", Some(roboto_id), 64.0, None);
title.color = [1.0, 0.5, 0.0, 1.0]; // Orange

// Use default system font by passing None
let debug_info = textware.create_text("FPS: 60", None, 14.0, None);
```

### 4. Layout & Sizing

Control wrapping and boundaries.

```rust
use textware::Wrap;

// Constrain width to 400px, auto height
textware.set_size(&mut title, Some(400.0), None);

// Enable word wrapping
textware.set_wrap(&mut title, Wrap::Word);
```

### 5. Render Loop

1.  **Prepare**: Uploads new glyphs to the GPU atlas.
2.  **Generate**: Creates the vertex mesh.
3.  **Draw**: Binds the texture and draws the mesh.

```rust
// 1. Prepare Cache (Call once per frame)
textware.prepare(&queue);

// 2. Generate Mesh
let mesh = textware.generate_mesh(&mut title);

// 3. Write to wgpu buffers
queue.write_buffer(&vertex_buffer, 0, bytemuck::cast_slice(&mesh.vertices));
queue.write_buffer(&index_buffer, 0, bytemuck::cast_slice(&mesh.indices));

// 4. Draw
{
    let mut rpass = encoder.begin_render_pass(...);
    rpass.set_pipeline(&text_pipeline);
    
    // Bind the Glyph Atlas (provided by textware)
    rpass.set_bind_group(0, textware.get_bind_group(), &[]);
    
    rpass.draw_indexed(0..mesh.indices.len() as u32, 0, 0..1);
}
```

## Shader Integration

The glyph atlas is an **R8Unorm** texture. The glyph coverage is stored in the **Red** channel.

**Vertex Structure:**
```rust
#[repr(C)]
struct TextVertex {
    position: [f32; 3],
    uv:       [f32; 2],
    color:    [f32; 4],
}
```

**WGSL Fragment Shader:**

```wgsl
@group(0) @binding(0) var t_atlas: texture_2d<f32>;
@group(0) @binding(1) var s_atlas: sampler;

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    // Read alpha from the Red channel
    let alpha = textureSample(t_atlas, s_atlas, in.uv).r;
    
    // Apply text color
    return vec4<f32>(in.color.rgb, in.color.a * alpha);
}
```