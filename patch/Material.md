## Material File Format
An EternityWorks material asset file minimally consists of the following framework:
```json
{
    "shader": "Shader Asset Path.sfn (Or UUID)",
    "uniforms": [

    ]
}
```
In addition, materials may also specify certain metadata. Currently, the supported metadata tags are:
- "isTranslucent": If true, the material will be forced to use the sequential render queue for the output render batch, instead of the standard queue.

## Material Fields
The defaults of the material instance's field values may be supplied through each member of the "uniforms" array, as seen in the example above. Furthermore, the engine is designed to parse different field types and extract the required data for the value. A normal field specifier will look something like this:
```json
{
    "name": "MyFieldName",
    "value": "Something..."
}
```
There is also an optional requirement to specify the field type, although this will generally not be required for most fields. The following is a list of all special field types, interactions and other notes that one should take care of when manually authoring materials.

#### Textures
Specifies a single texture to be referenced via the material instance. Corresponds to a `uint` in the actual shader struct declaration.

Example syntax (JSON):
```json
{
    "name": "diffuseTexture",
    "value": "Path/To/Texture.png, or UUID"
}
```
Example syntax (Slang):
```cpp
struct MaterialInstance
{
    uint diffuseTexture;
}

[[vk::binding(0, 1)]] Texture2D sprites[BINDLESS_LENGTH];
[[vk::binding(1, 1)]] SamplerState samplers[SAMPLERS_LENGTH];
float4 SampleTextureAt0(MaterialInstance instance)
{
    return sprites[NonUniformResourceIndex(instance.diffuseTexture)].Sample(samplers[0], float2(0.0f, 0.0f));
}
```
**Notes**:
- If the name of the field is `sprites` and the material is used in a `Renderer.DrawSprite` call, the engine will internally create a sub-material from the original material asset with the `sprites` value replaced with the 'sprite' argument used in the function.
- The value `ef6d6e45-6e68-4a64-8d9f-c66632ae8554` may be used to reference a 1x1 white pixel.
- The value `NullTexture` may be used to reference the default empty texture.

#### Sampler States
Specifies a single sampler state to be referenced via the material instance. Corresponds to a `uint` in the actual shader struct declaration.

Currently, the acceptable values for the 'value' field are:
- "PointClamp"
- "PointWrap"
- "LinearClamp"
- "LinearWrap"
Custom sampler states will be made available in the future.

Example syntax (JSON):
```json
{
    "name": "samplerID",
    "value": "PointClamp"
}
```
Example syntax (Slang):
```cpp
struct MaterialInstance
{
    uint samplerID;
}

[[vk::binding(0, 1)]] Texture2D sprites[BINDLESS_LENGTH];
[[vk::binding(1, 1)]] SamplerState samplers[SAMPLERS_LENGTH];
float4 SampleTextureAt0(MaterialInstance instance)
{
    return sprites[NonUniformResourceIndex(someTextureID)].Sample(samplers[NonUniformResourceIndex(instance.samplerID)], float2(0.0f, 0.0f));
}
```
**Notes**:
- The four primary sampler states are always inserted into the first four indices of the samplers array. If you wish to write a shader that uses a sampler but do not want it to be changeable by material, you may use the slang constant values of the same name as the sampler to index into the array.

#### Vectors
Specifies a 2, 3 or 4 point vector to be referenced via the material instance. Corresponds to either a `float2`, `float3` or `float4` in the actual shader struct declaration.

Example syntax (JSON):
```json
{
    "name": "A",
    "value": {
        "X": 0.0,
        "Y": 1.0
    }
},
{
    "name": "B",
    "value": {
        "X": 1.0,
        "Y": 2.0,
        "Z": 3.0
    }
},
{
    "name": "C",
    "value": {
        "X": 4.0,
        "Y": 5.0,
        "Z": 6.0,
        "W": 7.0
    }
}
```
Example syntax (Slang):
```cpp
struct MaterialInstance
{
    float2 A;
    float3 B;
    float4 C;
}
```

#### Compressed Vectors
Material files also support compressing a 2-component vector into a single 32-bit uint, where both components are expected to be 16-bit integers. The X component will therefore be compressed as the left 16 bits of the resulting uint, while the Y component will be the right 16 bits.

Example syntax (JSON):
```json
{
    "name": "myCompressedVector2",
    "type": "packed",
    "value": {
        "X": 255,
        "Y": 3000
    }
}
```
Example syntax (Slang):
```cpp
struct MaterialInstance
{
    uint myCompressedVector2;
}
float2 UncompressVector(uint compressedVector2)
{
    //Perform bit shifting and masking for each component respectively.
    return float2(compressedVector2 >> 16, compressedVector2 & 0xFFFF);
}
```

#### Texture Size
You may require to query the size of a supplied texture asset, either as a 2-component vector by itself, or packed into a uint like a compressed vector.

Example syntax (JSON):
```json
{
    "name": "textureSize",
    "type": "sizeof",
    "value": "SomeTexture1.png"
},
{
    "name": "textureSizePacked",
    "type": "packedsizeof",
    "value": "SomeTexture2.jpg"
}
```
Example syntax (Slang):
```cpp
struct MaterialInstance
{
    float2 textureSize;
    uint textureSizePacked;
}
```

**Notes**:
- This utility type will be deprecated eventually, since texture sizes will in time be available with the bindless arrays themselves, thereby allowing you to query the texture size through the texture ID itself.

#### Floats and Integers
Specifies a single floating-point value or integer value to be supplied to the material instance.

Example syntax (JSON):
```json
{
    "name": "someInt",
    "value": -320
},
{
    "name": "someFloat",
    "value": 99.999
}
```
Example syntax (Slang):
```cpp
struct MaterialInstance
{
    int someInt;
    float someFloat;
}
```

#### Colors
You can encode a hexadecimal color into a packed 32-bit uint as well. Both capitalised and non-capital hex digits are supported, as long as the string value itself is either 6 or 8 characters long. In the case of the latter, the final 2 digits refer to the alpha value of the color.

Example syntax (JSON):
```json
{
    "name": "colorValue",
    "type": "color",
    "value": "FF00FF"
},
{
    "name": "translucentColorValue",
    "type": "color",
    "value": "ffea4080"
}
```
Example syntax (Slang):
```cpp
struct MaterialInstance
{
    uint colorValue;
    uint translucentColorValue;
}
void UseColor(MaterialInstance instance)
{
    float4 unpacked = UnpackRGBInt(instance.colorValue);
    //do something with it...
}
```

#### Arrays
EternityWorks materials also let you specify fixed-size arrays of scalar types (EG: integers and floats.)

Example syntax (JSON):
```json
{
    "name": "arrayOfFiveFloats",
    "value": [0.01, 0.25, 3, 0.4, 0.9999]
}
```
Example syntax (Slang):
```cpp
struct MaterialInstance
{
    float[5] arrayOfFiveFloats;
}
```

#### Padding
The GPU will require struct instances passed to it to conform to a certain alignment. By default, this more-or-less corresponds to the standard alignment of structs in C and C#. As a result, EternityWorks will attempt to automatically assign the size and offset of each material field, but the algorithm may not be entirely accurate.

In lieu of this, you may also insert custom user-specified offsets in between material fields. This is done through the `bufferOffset` value, which while similarly enclosed in a JSON object, will override its requirement for a 'value' or 'name' property entirely.
```json
{
    "name": "SomeVec3",
    "value": {
        "X": 5.0,
        "Y": 0.0,
        "Z": -1.24
    }
},
{
    "bufferOffset": 4
},
{
    "name": "SomeVec2":
    "value": {
        "X": 0.0,
        "Y": 0.0
    }
}
```
For more information about GPU buffer alignment, see the [shader memory layout rules for Vulkan](https://docs.vulkan.org/guide/latest/shader_memory_layout.html).