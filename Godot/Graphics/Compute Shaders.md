# Overall Flow
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
Can be done by simply writing `pipeline = rd.compute_pipeline_create(shader_rid)`

## Create / assign buffers