# GDScript Setup
Godot's compute shader is very C like, where you have to pass in `RID`s if you want to operate on it.
Compute shader in Godot 4 can be used for general calculation purposes as well as for creating compositor effects.
## Create / Get Rendering Device
The rendering device is of type `RenderingDevice`.
It can be created with `RenderingServer.create_local_rendering_device()` for creating a separate rendering device than the main one, or with `RenderingServer.get_rendering_device()`, which will return the main rendering device of the game / instance.
## Create the Shader SPIRV
If we're loading the shader from a file, the we can get the SPIRV with:

```gdscript
var rd: RenderingDevice
var shader_rid: RID

func _init() -> void:
	# create / get a rendering device here

	var shader_file: Resource = load("PATH/TO/SHADERFILE/shader_filename.glsl")
	var shader_spirv: RDShaderSpirv = shader_file.get_spirv()
	
	shader_rid = rd.shader_create_from_spirv()
```

Otherwise, if we store the shader contents on a string variable, we can do it this way:

```gdscript
var rd: RenderingDevice
var shader_rid: RID

func _init() -> void:
	# Create / get a rendering device here

	var shader_source: RDShaderSource = RDShaderSource.new()
	shader_source.set_stage_source(
		RenderingDevice.SHADER_STAGE_COMPUTE, 
		"Shader string goes here"
	)
	
	# Create SPIRV
	var spirv: RDShaderSPIRV = rd.shader_compile_spirv_from_source(shader_source)
	if spirv.compile_error_compute.is_empty():
		shader_rid = rd.shader_create_from_spirv(spirv)
```

I am setting the shader string inside another class like this:

```gdscript
class_name GerstnerCompute extends Node


static var use_compute: bool = true

const COMPUTE_CODE: String = """
#version 450

...

"""

# I can call this function whenever I need to get the shader code. I just need to supply the parameter and the compute shader code will be updated automatically.'
static func get_compute_code(array_size: int) -> String:
	return COMPUTE_CODE % [array_size, array_size, array_size, array_size, array_size]
```
## Create Shader Pipeline
Can be done by simply writing `pipeline = rd.compute_pipeline_create(shader_rid)`.

## Create / Assign Buffers
Buffers can be of two type (at least what I've used so far): texture and storage buffers. Texture are for, well, textures and storage buffers are for numbers (floats, integers, array of floats or integers, they should all be stored within a `PackedFloat32Array`). They can be used for both read and write.
### Storage buffers

I like making a function for creating the contents of the buffer. For example, in Sabitah, I made a function to get the wave data and transform it into a `PackedFloat32Array`:

```gdscript
func _get_wave_data() -> PackedByteArray:
	return PackedFloat32Array([# Wave settings
		ocean_manager.wave_normal_amount, ocean_manager.amplitude_scale, ocean_manager.frequency_scale, ocean_manager.time_scale,
		ocean_manager.steepness_e0, ocean_manager.steepness_e1, ocean_manager.cpu_time
	]).to_byte_array()
```

And then I can call this function during initialization of the buffer:

```gdscript
var wave_buffer_rid: RID

func _init() -> void:
	# ...
	
	var wave_data: PackedByteArray = _get_wave_data()
	wave_buffer_rid = rd.storage_buffer_create(wave_data.size(), wave_data)
	
	# ...
```

Or I can call it during each frame, to update the values inside the buffer:

```gdscript
func _process(delta: float) -> void:
	# ...

	var wave_data: PackedByteArray = _get_wave_data()
	rd.buffer_update(wave_buffer_rid, 0, wave_data.size(), wave_data)
	
	# ...
```

### Texture Buffers
Additional information needs to be provided for creating texture buffers. On top of setting up the buffer, you need to provide the texture format, including the texture width and height, the data format, and the usage bits. If you want the compute shader to read a texture from an existing file (for example, this is how I read a wave texture in Sabitah), you can set it up this way:

```gdscript
func _init() -> void:
	# ...

	var texture_view: RDTextureView = RDTextureView.new()
	var wave_image: Image = ocean_manager.wave_texture.get_image()
	wave_image.convert(Image.FORMAT_RGBAF)

	var texture_format: RDTextureFormat = RDTextureFormat.new()
	texture_format.width = ocean_manager.wave_texture.get_width()
	texture_format.height = ocean_manager.wave_texture.get_height()
	texture_format.format = RenderingDevice.DATA_FORMAT_R32G32B32A32_SFLOAT
	texture_format.usage_bits = (
		RenderingDevice.TEXTURE_USAGE_STORAGE_BIT + 
		RenderingDevice.TEXTURE_USAGE_SAMPLING_BIT
	)

	wave_tex_rid = rd.texture_create(texture_format, texture_view, [wave_image.get_data()])
	
	# ...
```

You can just call this one time during initialization of the buffer.

## Create Uniform and Uniform Set
If we want to send data from the CPU to the GPU, we need to use uniforms. The uniform will need to be setup from the CPU, and needs to be updated on every frame (if necessary, otherwise one time during initialization is fine too). Setting up uniforms in the shader file is straightforward:

```glsl
...

layout(set = 0, binding = 3, std430) restrict buffer WaveData {
	...
} wd;

layout(set = 0, binding = 4, rgba32f) restrict readonly uniform image2D wave_tex;

...
```

And then you need to setup the uniform with matching binding integer value:

```gdscript
func _process(delta:float) -> void:
	# ...

	# Storage buffer
	var wave_uniform: RDUniform = RDUniform.new()
	wave_uniform.uniform_type = RenderingDevice.UNIFORM_TYPE_STORAGE_BUFFER
	wave_uniform.binding = 3 # change accordingly
	wave_uniform.add_id(wave_buffer_rid)

	# Texture buffer
	var wave_tex_uniform: RDUniform = RDUniform.new()
	wave_tex_uniform.uniform_type = RenderingDevice.UNIFORM_TYPE_IMAGE
	wave_tex_uniform.binding = 4 # change accordingly
	wave_tex_uniform.add_id(wave_tex_rid)
	
	# ...
```

After that we need to create a uniform set to group our uniforms together. Also provide the `shader_rid` that we've created previously and the uniform set according to what we have in the shader:

```gdscript
var uniform_set: RID

func _process(delta: float) -> void:

	# create uniform set
	uniform_set = rd.uniform_set_create([
		..., wave_uniform, wave_tex_uniform, ...
	], shader_rid, 0) # set = 0
	
	# execute shader
	var compute_list: int = rd.compute_list_begin()
	rd.compute_list_bind_compute_pipeline(compute_list, pipeline)
	rd.compute_list_bind_uniform_set(compute_list, uniform_set, 0)
	rd.compute_list_dispatch(compute_list, 1, ocean_manager.wave_image_cache.get_height(), 1)
	rd.compute_list_end()

```

## Getting the Data from Compute Shader
If your shader runs every frame, you will need to call the `rd.compute_list_dispatch` function from above every frame as well. After that, you need to synchronize the CPU with the GPU before getting the data:

```gdscript
func _process(delta: float) -> void:
	# ...
	# Call the compute
	
	# Sync with GPU
	rd.submit()
	rd.sync()
```

And then you can get the data from the buffer through the rendering device:

```gdscript

func _process(delta: float) -> void:
	# ...
	# Call the compute
	# Sync with GPU

	# Get the data
	# Remember that the result will always be a PackedFloat32Array
	var output_bytes: PackedByteArray = rd.buffer_get_data(target_buffer_rid)
	var out: PackedFloat32Array = output_bytes.to_float32_array()

```

## Cleanup RIDs
Since `RID`s in Godot is not reference counted, we need to remove it on our own after we're done with them to avoid memory leak:

```gdscript
func _exit_tree() -> void:
	if rd == null:
		return
	
	rd.free_rid(pipeline)
	pipeline = RID()
	
	# ... and the rest of the RID
```