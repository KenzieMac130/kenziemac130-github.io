# Extending MTL with Subsurface Scattering

### Version 1.0.0
This post proposes an extension to MTL to transmit subsurface scattering parameters across applications.

# Introduction
MTL is a highly extendable text-based material file format that is supported by many applications. MTL is also a very old file format that was defined in a relatively early stage of computer graphics. Over the years different applications have introduced extensions, such as emission support or even PBR support, to keep up with the needs of modern computer graphics. 

Subsurface scattering is an important component to rendering believable organic materials such as foliage, food, and characters. Subsurface scattering simulates the way light bounces around and is absorbed inside of an object's volume. This important rendering feature is currently not properly defined in MTL.

In order to create this proposal, a wide variety of offline renderers and game engines were investigated to better understand common features and limitations. Upon investigation, it became clear that each had different options and techniques when approaching subsurface scattering. However, a common set of artistic parameters to control the general results of the final image were present. This patch focuses on transmitting those parameters via MTL. It also became clear that this proposal should only focus on homogenous volumes. It is not standard among various renderers to support non-homogenous subsurface scattering, such as layered physical materials or volumetric textured interiors. Renderers that do support some form of non-homogenous subsurface scattering vary too widely in approach to find significant common ground. In findings, many real-time programs were fundamentally incapable of non-homogenous subsurface scattering, as they "baked" surface profiles into look-up tables. Some applications also allow specifying another source for the surface's color when using subsurface scattering. This proposal suggests that the primary subsurface color should always be overruled by the "Kd" parameter to ensure compatibility with software that doesn't support loading subsurface scattering.

In order to achieve these goals, three new parameters will be introduced: "Sf", "map_Sf", and "Sr". "Sf" is the subsurface factor that blends between a diffuse shading model and a subsurface scattering shading model. "map_Sf" is the texture map for the "Sf" parameter. "Sr" is a subsurface radius vector which defines how far light is allowed to scatter through the medium. This convention was chosen after investigating existing support for transmission using the "Tr" and "Tf" parameters for transmission factor and transmission filter.

# Specification

## Prerequisites
* The application **must** be compliant with the [1995 MTL specification by Alias/Wavefront](http://paulbourke.net/dataformats/mtl/) when loading the parameters defined by this extension.
* Any vendor specific non-compliance in files **must** be ignored by applications by default.
* Applications **may** provide compatibility options to users if loading non-compliant files is deemed necessary.
* The application **must** get the visible primary color of the surface from the "Kd" or "map_Kd" parameters.

## Sf: Subsurface Factor
* This parameter blends between a diffuse shading model and a subsurface scattering shading model.
* This parameter **must** be a scalar value.
* "0.0" **must** represent a fully diffuse shading model. 
* "1.0" **must** represent a fully subsurface shading model.
* Any value outside of the "0.0" through "1.0" range is invalid and **must** be clamped by the application.
### Examples
```
Sf 1.0
```

## map_Sf: Subsurface Factor Texture
* This texture parameter blends between a standard diffuse shading model and a subsurface scattering shading model.
* The texture data **must** be read as a scalar value.
* "0.0" **must** represent a fully diffuse shading model. 
* "1.0" **must** represent a fully subsurface shading model.
* Any value outside of the "0.0" through "1.0" range is invalid and **must** be clamped by the application.
### Examples
```
map_Sf -imfchan l Textures/CharacterSkinMask.png
```

## Sr: Subsurface Radius
* This parameter defines how far light is allowed to scatter through the medium.
* This parameter **must** be a three component vector value or a valid spectral file path.
* The vector **must** map to "RGB" channels.
* The vector **must** be represented in meters.
* The color representation **must** be in linear space.
* Any value below "0.0" on any channel **must** be clamped by the application.
* This parameter **can** define a spectral curve file path using the "spectral" keyword.
* The application **must not** apply subsurface scattering to the material if it is unable to properly utilize the spectral file.
### Examples
```
Sr 2.0 0.4 0.2
```
```
Sr spectral MilkScatter.rfl
```

### Conversion to Color/Distance
* When writing: The application **must** follow this formula to convert a color, distance pair into the radius vector as necessary\:

``` glsl
/* Convert subsurface color, and distance into a radius vector. */
vec3 sss_radius = sss_color.rgb * sss_distance;
```
    
* When reading: The application **must** follow this formula to convert the radius vector into a color, distance pair as necessary\:

``` glsl
/* Convert radius vector into subsurface color, and distance scalar.
*  FLOAT_EPSILON is a tiny number to prevent divide by zero errors.
*  It is acceptable to replace FLOAT_EPSILON with 1.0 as seen fit. */
float sss_distance = max(max(sss_radius.r, sss_radius.g), max(sss_radius.b, FLOAT_EPSILON)));
vec3 sss_color = sss_radius.rgb / sss_distance;
```

## map_Sr: Reserved
* This parameter is reserved for possible future use as the texture component to "Sr".
* Applications **must not** attempt to load this parameter.
* Applications **must not** attempt to write this parameter.
