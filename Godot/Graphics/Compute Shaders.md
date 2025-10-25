# GDScript Setup
Godot's compute shader is very C like, where you have to pass in `RID`s if you want to operate on it.
Compute shader in Godot 4 can be used for general calculation purposes as well as for creating compositor effects.
## Create / get rendering device
The rendering device is of type `RenderingDevice`.
It can be created with `RenderingServer.create_local_rendering_device()` for creating a separate rendering device than the main one, or with `RenderingServer.get_rendering_device()`, which will return the main rendering device of the game / instance.
## Create the shader SPIRV
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

## Create shader pipeline
Can be done by simply writing `pipeline = rd.compute_pipeline_create(shader_rid)`.

## Create / assign buffers
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