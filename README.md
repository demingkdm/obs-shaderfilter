# obs-shaderfilter

## Introduction

The obs-shaderfilter plugin for [OBS Studio](http://obsproject.com/) is intended to allow users to apply 
their own shaders to OBS sources. This theoretically makes possible some simple effects like drop shadows
that can be implemented strictly in shader code. 

Please note that this plugin may expose a reasonable number of bugs in OBS, as it uses the shader parser and 
the property system in somewhat unusual ways. It should be considered to be in a prerelease state at this time.

## Installation

The binary package mirrors the structure of the OBS Studio installation directory, so you should be able to
just drop its contents alongside an OBS Studio install (usually at C:\Program Files (x86)\obs-studio\). The 
necessary files should look like this: 

    obs-studio
    |---data
    |   |---obs-plugins
    |       |---obs-shaderfilter
    |           |---examples
    |               |---blink.shader
    |               |---border.shader
    |               |---drop_shadow.shader
    |               |---filter-template.effect
    |               |---multiply.shader
    |               |---pulse.effect
    |               |---rectangular_drop_shadow.shader
    |               |---rounded_rect.shader
    |           |---locale
    |               |---en-US.ini
    |---obs-plugins
        |---32bit
        |   |---obs-shaderfilter.dll
        |---64bit
            |---obs-shaderfilter.dll

## Usage

The filter can be added to any source through the "Filters" option when right-clicking on a source. The name
of the filter is "User-defined shader." 

Shaders can either be entered directly in a text box in the filter properties, or loaded from a file. To change 
between the two modes, use the "Load shader text from file" toggle. If you are entering your shader text directly,
note that you will need to use the "Reload effect" button to apply your changes. This can also be used to reload an
external file if changes have been made. OBS shaders appear to be written in HLSL, but I'm not completely sure.

The option is provided to render extra pixels on each side of the source. This is useful for effects like shadows
that need to render outside the bounds of the original source. 

Normally, all that's required for OBS purposes is a pixel shader, so the plugin will wrap your shader text with a 
standard template to add a basic vertex shader and other boilerplate. If you wish to customize the vertex shader
or other parts of the effect for some reason, you can check the "Override entire effect" option. 

Any parameters you add to your shader (defined as `uniform` variables) will be detected by the plugin and exposed
in the properties window to have their values set. Currently, only `int`, `float`, `bool`, `texture2d`, and `float4`
parameters are supported. (`float4` parameters will be interpreted by the properties window as colors.) Note that the 
current version of OBS has a [bug](https://obsproject.com/mantis/view.php?id=1028) that prevents filter properties
from refreshing when filter data changes; if you update the parameters in your shader, you will need to close and
reopen the filters window to see them. 

Note that if your shader has syntax errors and fails to compile, OBS does not provide any error messages; you will
simply see your source render nothing at all. In many cases the output of the effect parser will be written to the
OBS log file, which you can view with the Help -> Log Files menu in OBS. 

### Standard parameters

The plugin automatically populates a few parameters which your shader can use. If you choose to override the entire
effect, be sure to define these as `uniform` variables and use them where necessary. (The filter should gracefully 
handle these variables being missing, but the shader may malfunction.)

* **`ViewProj`** (`float4x4`)&mdash;The view/projection matrix. (Standard for all OBS filters.)
* **`image`** (`texture2d`)&mdash;The image to which the filter is being applied, either the original output of 
  the source or the output of the previous filter in the chain. (Standard for all OBS filters.)
* **`elapsed_time`** (`float`)&mdash;The time in seconds which has elapsed since the filter was created. Useful for 
  creating animations. 
* **`uv_offset`** (`float2`)&mdash;The offset which should be applied to the UV coordinates of the vertices. This is
  used in the standard vertex shader to draw extra pixels on the borders of the source.
* **`uv_scale`** (`float2`)&mdash;The scale which should be applied to the UV coordinates of the vertices. This is 
  used in the standard vertex shader to draw extra pixels on the borders of the source.
* **`uv_pixel_interval`** (`float2`)&mdash;This is the size in UV coordinates of an individual texel. You can use
  this to convert the UV coordinates of the pixel being processed to the coordinates of that texel in the source
  texture, or otherwise scale UV coordinate distances into texel distances.
  
### Example shaders

Several examples are provided in the plugin's *data/examples* folder. These can be used as-is for some hopefully
useful common tasks, or used as a reference in developing your own shaders. Note that the *.shader* and *.effect* 
extensions are for clarity only, and have no specific meaning to the plugin. Text files with any extension can be
loaded.

* *blink.shader*&mdash;A shader that fades the opacity of the output in and out over time, with a configurable speed
  multiplier. Demonstrates the user of the `elapsed_time` parameter.
* *border.shader*&mdash;A shader that adds a solid border to all extra pixels outside the bounds of the input. 
* *drop_shadow.shader*&mdash;A shader that adds a basic drop shadow to the input. Note that this is done with a simple
  uniform blur, so it won't look quite as good as a proper Gaussian blur. This is also an O(N&sup2;) blur on the size 
  of the blur, so be very conscious of your GPU usage with a large blur size.
* *filter_template.effect* (Overrides entire effect)&mdash;A copy of the default effect used by the plugin, which simply
  renders the input directly to the output after scaling UVs to reflect any extra border pixels. This is useful as a starting
  point for developing new effects, especially those that might need a custom vertex shader. (Note that modifying this file will
  not affect the internal effect template used by the plugin.)
* *multiply.shader*&mdash;A shader that multiplies the input by another image specified in the parameters. Demonstrates the use 
  of user-defined `texture2d` parameters.
* *pulse.effect* (Overrides entire effect)&mdash;An effect that varies the size of the output over time. This demonstrates 
  a custom vertex shader that manipulates the position of the rendered vertices based on user data. Note that moving the vertices 
  in the vertex shader will not affect the logical size of the source in OBS, and this may mean that pixels outside the source's
  bounds will get cut off by later filters in the filter chain.
* *rectangular_drop_shadow.shader*&mdash;A shader that renders an optimized drop shadow for sources that are opaque and rectangular. 
  Pixels inside the bounds of the input are treated as solid; pixels outside are treated as opaque. The complexity of the blur
  does not increase with its size, so you should be able to make your blur size as large as you like wtihout affecting
  GPU load. 
* *rounded_rect.shader*&mdash;A shader that rounds the corners of the input, optionally adding a border outside the rounded 
  edges.

## Building

If you wish to build the obs-shaderfilter plugin from source, you should just need [CMake](https://cmake.org/) 
and the OBS Studio libraries and headers.

* [obs-shaderfilter source repository](https://github.com/nleseul/obs-shaderfilter)
* [OBS Studio source repository](https://github.com/jp9000/obs-studio)

I don't believe that the OBS project provides prebuilt libraries; you're probably going to have the best luck
building your own OBS binaries from the source. Refer to the OBS repository for more information on that.

When building in CMake, the OBSSourcePath configuration value should refer to the libobs subfolder 
in the OBS source release. The build pipeline will look for headers in this location, and for libraries
in a "build" folder relative to that path (where the OBS build process puts them). 

Installation logic is provided through CMake as well; you can set the CMAKE_INSTALL_PREFIX configuration value
to choose the folder to which the files will be copied. You can also manually copy all files to the locations
described above.

## License

This project is licensed under the "[Unlicense](http://unlicense.org/)", because copy[right|left] is a hideous
mess to deal with and I don't like it. 
