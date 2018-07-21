Physically-Based Rendering in glTF 2.0 using WebGL
==================================================

[![](images/BoomBox.JPG)](http://github.khronos.org/glTF-WebGL-PBR/)

This is a raw WebGL demo application for the introduction of physically-based materials to the core glTF 2.0 spec. This project is meant to be a barebones reference for developers looking to explore the widespread and robust capabilities of these materials within a WebGL project that isn't tied to any external graphics libraries. For a DirectX sample please head over to [this repo](https://github.com/Microsoft/glTF-DXViewer) instead.

这是一个原始的WebGL演示应用程序，用于将基于物理的材料引入到glTF 2.0核心规范。 该项目旨在成为希望探索WebGL项目中这些材质的广泛且强大功能的开发人员的准系统参考，该WebGL项目不受任何外部图形库的束缚。

If you would like to see this in action, [view the live demo](http://github.khronos.org/glTF-WebGL-PBR/).

> **Controls**
>
> `click + drag` : Rotate model
>
> `scroll` : Zoom camera
>
> `GUI` : Use to change models

Physically-Based Materials in glTF 2.0
--------------------------------------

With the change from glTF 1.0 to glTF 2.0, one of the largest changes included core support for materials that could be used for physically-based shading. Part of this process involved chosing technically accurate, yet user-friendly, parameters for which developers and artists could use intuitively. This resulted in the introduction of the **Metallic-Roughness Material** to glTF. If you would like to read more about glTF, you can find the content at its [GitHub page](https://github.com/KhronosGroup/glTF), but I will take a bit of time to explain how this new material works.

随着glTF 1.0标准到glTF 2.0标准的变化，最大的变化之一是对基于物理着色的材质的核心支持。这一过程的一部分涉及选择技术上准确，对用户友好的参数，开发人员和艺术家可以直观地使用这些参数。这导致了将Metallic-Roughness Material引入glTF。如果您想了解更多关于glTF的信息，您可以在其GitHub页面上找到这些内容，但是我会花一点时间来解释这种新材料是如何工作的。

A surface using the Metallic-Roughness material is governed by three parameters:

使用Metallic-Roughness 材质的表面由三个参数控制：
* `baseColor` - The inherent color attribute of a surface  物体表面的基本颜色属性
* `metallic` -  A float describing how metallic the surface is  描述物体表面金属属性的浮点数
* `roughness` - A float describing how rough the surface is  描述物体表面粗糙程度的浮点数

These parameters can be provided to the material in two ways. Either the parameters can be given constant values, which would dictate the shading of an entire mesh uniformly, or textures can be provided that map varying values over a mesh. In this project, all of the glTF files followed the latter case. It is important to note here that although `metallic` and `roughness` are separate parameters, they are provided as a single texture in which the `metallic` values are in the blue channel and the `roughness` values are in the green channel to save on space.

这些参数可以通过两种方式提供给材质。要么参数可以是常量，这将导致整个网格采用统一的着色方式，也可以提供在网格上映射变化值的纹理。在这个项目中，所有的glTF文件都遵循后一种情况。这里需要注意的是，虽然金属和粗糙度是单独的参数，但它们是作为单一纹理提供的，其中金属值位于蓝色通道中，而粗糙度值位于绿色通道中从而节省空间。

**Base Color of a Boombox**

<img src="models/BoomBox/glTF/BoomBox_baseColor.png" width="300" height="300"/> -> <img src="images/BoomBox-baseColor.JPG" width="300" height="300"/>

**Metallic-Roughness of a Boombox**

<img src="models/BoomBox/glTF/BoomBox_occlusionRoughnessMetallic.png" width="300" height="300"/> -> <img src="images/BoomBox-metallicRoughness.JPG" width="300" height="300"/>

Although these are the core parameters of the Metallic-Roughness material, often a user will want to provide additional maps for features such as normals, ambient occlusion, or emissiveness. Similarly to above, these are usually provided as a texture that corresponds to the parts of the mesh that have shifted normals, are occluded and/or are emissive, respectively. However, since these are not a part of the Metallic-Roughness material itself, they are provided as a separate portion to the material.

虽然这些是Metallic-Roughness材质的核心参数，但用户往往希望为法线，环境遮挡或发射等特征提供附加的贴图。与上面类似，这些通常是作为纹理来提供的，这些纹理表示网格外观具有偏移法线，环境遮挡，发光度等属性。但是，由于这些不是Metallic-Roughness材质本身的一部分，它们作为材料的单独部分提供。

The overall structure of a material would then look something like this in glTF 2.0:

在glTF 2.0中，材质的整体结构会看起来像这样：

```
"materials": [
  {
    "pbrMetallicRoughness": {
      "baseColorTexture": {...},
      "metallicRoughnessTexture": {...}
    },
    "normalTexture": {...},
    "occlusionTexture": {...},
    "anyOtherAttribute": {...},
    "name": "myMetallicRoughnessMaterial"
  }
]
```

Using Metallic-Roughness to Shade  使用Metallic-Roughness来进行着色处理
----------------------------------

Once we have read in these values and passed them into the fragment shader correctly, we need to compute the final color of each fragment. Without going too far into the theory behind PBR, this is how this demo application computes the color.

一旦我们读入这些值并将它们正确传递到片段着色器中，我们需要计算每个片段的最终颜色。没有太深入PBR的理论，这个应用程序主要演示如何计算颜色。

It is first important to choose a microfacet model to describe how light interacts with a surface. In this project, I use the [Cook-Torrance Model](https://web.archive.org/web/20160826022208/https://renderman.pixar.com/view/cook-torrance-shader) to compute lighting. However, there is a large difference between doing this based on lights within a scene versus an environment map. With discrete lights, we could just evaluate the BRDF with respect to each light and average the results to obtain the overall color, but this is not ideal if you want a scene to have complex lighting that comes from many sources.

首先选择微平面模型来描述光与表面的相互作用是非常重要的。在这个项目中，我使用[Cook-Torrance Model]模型来计算照明。然而，基于场景内的灯光进行渲染与基于环境贴图进行渲染两者差距很大。使用离散光源，我们可以对每个光源评估BRDF，并对结果进行平均以获得整体色彩，但如果您希望场景具有来自多个来源的复杂光照，则这并不理想。

### Environment Maps  环境贴图

This is where environment maps come in! Environment maps can be thought of as a light source that surrounds the entire scene (usually as an encompassing cube or sphere) and contributes to the lighting based on the color and brightness across the entire image. As you might guess, it is extremely inefficient to assess the light contribution to a single point on a surface from every visible point on the environment map. In offline applications, we would typically resort to using importance sampling within the render and just choose a predefined number of samples. However, as described in [Unreal Engine's course notes on real-time PBR](http://blog.selfshadow.com/publications/s2013-shading-course/karis/s2013_pbs_epic_notes_v2.pdf), we can reduce this to a single texture lookup by baking the diffuse and specular irradiance contributions of the environment map into textures. You could do this yourself as described in the course notes, but there is also a resource called [IBL Baker](http://www.derkreature.com/iblbaker/) that will create these textures for you. The diffuse irradiance can be stored in a cube map, however, we expect the sharpness of specular reflection to diminish as the roughness of the object increases. Because of this, the different amounts of specular irradiance can be stored in the mip levels of the specular cube map and accessed in the fragment shader based on roughness.

这是环境贴图进来的地方！环境贴图可以被认为是围绕整个场景的光源（通常是立方体或球体），并且根据整个图像的颜色和亮度对光照做出贡献。正如您可能猜到的那样，从环境贴图上的每个可见点评估表面上单个点的光贡献效率非常低。在离线应用程序中，我们通常会在渲染中使用重要性采样，并选择预定义数量的采样。但是，如[Unreal Engine's course notes on real-time PBR]所述，我们可以将环境贴图的漫反射和镜面辐射贡献烘焙为纹理，通过单个纹理查找来提高效率。您可以按照课程笔记中所述自行完成此操作，但也有一个名为[IBL Baker]的资源，可以为您创建这些纹理。漫反射辐照度可以存储在立方体贴图中，但是，随着物体粗糙度的增加，我们预计镜面反射的清晰度会降低。因此，镜面辐照度可以存储在镜面立方体贴图的mip级别中，并根据粗糙度在片段着色器中进行访问。


**Diffuse Front Face**

![](textures/papermill/diffuse/diffuse_front_0.jpg)

**Specular Front Face**

![](textures/papermill/specular/specular_front_0.jpg) ![](textures/papermill/specular/specular_front_1.jpg) ![](textures/papermill/specular/specular_front_2.jpg) ![](textures/papermill/specular/specular_front_3.jpg) ![](textures/papermill/specular/specular_front_4.jpg) ![](textures/papermill/specular/specular_front_5.jpg) ![](textures/papermill/specular/specular_front_6.jpg) ![](textures/papermill/specular/specular_front_7.jpg) ![](textures/papermill/specular/specular_front_8.jpg) ![](textures/papermill/specular/specular_front_9.jpg)

### BRDF

At this point, we can pick out the diffuse and specular incoming light from our environment map, but we still need to evaluate the BRDF at this point. Instead of doing this computation explicitly, we use a BRDF lookup table to find the BRDF value based on roughness and the viewing angle. It is important to note that this lookup table changes depending on which microfacet model we use! Since this project uses the Cook-Torrance model, we use the following texture in which the y-axis corresponds to the roughness and the x-axis corresponds to the dot product between the surface normal and viewing vector.

此时，我们可以从环境贴图中获取漫反射和镜面反射的入射光，但我们仍然需要在此处评估BRDF。我们不使用明确的计算方法，而是使用BRDF查找表来查找基于粗糙度和视角的BRDF值。请注意，此查找表根据我们使用的微表面模型而变化！由于该项目使用Cook-Torrance模型，因此我们使用以下纹理，其中y轴对应于粗糙度，x轴对应于曲面法线向量和视点方向向量之间的点积。

![](textures/brdfLUT.png)

### Diffuse and Specular Color  漫反射和镜面反射颜色

We now have the diffuse and specular incoming light and the BRDF, but we need to use all the information we have gathered thus far to actually compute the lighting. Here is where the `metallic` and `baseColor` values come into play. Although the `baseColor` dictates the inherent color of a point on a surface, the `metallic` value tells us how much of the `baseColor` is represented in the final color as diffuse versus specular. For the diffuse color, we do this by interpolating between black and the base color based on the `metallic` value such that the diffuse color is closer to black the more metallic it is. Conversely, for the specular color, we interpolate such that the surface holds more of the `baseColor` the more metallic it is.

我们现在有漫反射和镜面反射的入射光和BRDF，但是我们需要使用迄今为止收集的所有信息来实际计算照明。这就是metallic和baseColor值有作用的地方。虽然baseColor规定了物体表面上点的固有颜色，但metallic值告诉我们在最终颜色计算中baseColor有多少表示为漫反射有多少表示为镜面反射。对于漫反射颜色，我们基于metallic值在黑色和基础颜色之间进行插值来实现这一点，metallic值越大使得漫反射颜色越接近黑色。相反，对于镜面颜色，我们进行插值，metallic值越大使得物体表面拥有更多的baseColor，而更具金属感。

### Final Color  最终颜色

Finally, we can compute the final color by summing the contributions of diffuse and specular components the color in the following manner:

最终，我们可以通过下列方式对漫反射和镜面反射分量的颜色进行求和来计算最终颜色：

`finalColor = (diffuseLight * diffuseColor) + (specularLight * (specularColor * brdf.x + brdf.y))`

Appendix  附录
------------

The core lighting equation this sample uses is the Schlick BRDF model from [An Inexpensive BRDF Model for Physically-based Rendering](https://www.cs.virginia.edu/~jdl/bib/appearance/analytic%20models/schlick94b.pdf)

本示例使用的核心照明方程是来自[An Inexpensive BRDF Model for Physically-based Rendering]的Schlick BRDF模型

```
vec3 specContrib = F * G * D / (4.0 * NdotL * NdotV);
vec3 diffuseContrib = (1.0 - F) * diffuse;
```

If you're familiar with implementing the phong model, you may think that the diffuse and specular contributions simply need to be summed up to obtain the final lighting. However, in the context of a BRDF, the diffuse and specular components are not accounting for the *energy* of the incident light, which can cause some confusion.

如果您熟悉phong模型的实现，您可能会认为漫反射和镜面反射分量只需要简单计算即可获得最终的照明效果。但是，在BRDF的背景下，漫反射和镜面反射分量并不能解释入射光的能量，这会造成一些混淆。<br><br>
Using a BRDF, the diffuse and specular parts describe the *bidirectional reflectance*, which we have to scale by the *energy* received from the light in order to obtain the final intensity that reaches the eye of the viewer (as outlined in the respective [paper by Cook and Torrance](http://graphics.pixar.com/library/ReflectanceModel/).

使用BRDF，漫反射和镜面反射部分描述了*双向反射*，我们必须通过从光线接收到的*能量*进行缩放，以获得到达观看者眼睛的最终强度（如[paper by Cook and Torrance]文中相应的概述）<br><br>
According to the basic cosine law (as described by [Lambert](https://archive.org/details/lambertsphotome00lambgoog)), the energy is computed using the dot product between the light's direction and the surface normal. Therefore, the final intensity that will be used for shading is computed as follows:

根据基本余弦定律（如[Lambert]文中所述），使用光线方向和表面法线之间的点积来计算能量。因此，将用于进行颜色着色的最终强度计算如下：
```
vec3 color = NdotL * u_LightColor * (diffuseContrib + specContrib);
```

Below here you'll find common implementations for the various terms found in the lighting equation.
These functions may be swapped into pbr-frag.glsl to tune your desired rendering performance and presentation.

在这里您可以找到照明方程中各种术语的常见实现。这些函数可以应用到pbr-frag.glsl中来调整渲染性能和渲染效果。

### Surface Reflection Ratio (F) 表面反射率（F）

**Fresnel Schlick**
Simplified implementation of fresnel from [An Inexpensive BRDF Model for Physically based Rendering](https://www.cs.virginia.edu/~jdl/bib/appearance/analytic%20models/schlick94b.pdf) by Christophe Schlick.<br>根据Christophe Schlick[An Inexpensive BRDF Model for Physically based Rendering]简化实现菲涅耳。

```
vec3 specularReflection(PBRInfo pbrInputs)
{
    return pbrInputs.metalness + (vec3(1.0) - pbrInputs.metalness) * pow(1.0 - pbrInputs.VdotH, 5.0);
}
```

### Geometric Occlusion (G) 几何遮挡

**Cook Torrance**
Implementation from [A Reflectance Model for Computer Graphics](http://graphics.pixar.com/library/ReflectanceModel/) by Robert Cook and Kenneth Torrance,

```
float geometricOcclusion(PBRInfo pbrInputs)
{
    return min(min(2.0 * pbrInputs.NdotV * pbrInputs.NdotH / pbrInputs.VdotH, 2.0 * pbrInputs.NdotL * pbrInputs.NdotH / pbrInputs.VdotH), 1.0);
}
```

**Schlick**
Implementation of microfacet occlusion from [An Inexpensive BRDF Model for Physically based Rendering](https://www.cs.virginia.edu/~jdl/bib/appearance/analytic%20models/schlick94b.pdf) by Christophe Schlick.<br>根据Christophe Schlick[An Inexpensive BRDF Model for Physically based Rendering]简化实现微表面遮挡。

```
float geometricOcclusion(PBRInfo pbrInputs)
{
    float k = pbrInputs.perceptualRoughness * 0.79788; // 0.79788 = sqrt(2.0/3.1415); perceptualRoughness = sqrt(alphaRoughness);
    // alternately, k can be defined with
    // float k = (pbrInputs.perceptualRoughness + 1) * (pbrInputs.perceptualRoughness + 1) / 8;

    float l = pbrInputs.LdotH / (pbrInputs.LdotH * (1.0 - k) + k);
    float n = pbrInputs.NdotH / (pbrInputs.NdotH * (1.0 - k) + k);
    return l * n;
}
```

**Smith**
The following implementation is from "Geometrical Shadowing of a Random Rough Surface" by Bruce G. Smith

```
float geometricOcclusion(PBRInfo pbrInputs)
{
  float NdotL2 = pbrInputs.NdotL * pbrInputs.NdotL;
  float NdotV2 = pbrInputs.NdotV * pbrInputs.NdotV;
  float v = ( -1.0 + sqrt ( pbrInputs.alphaRoughness * (1.0 - NdotL2 ) / NdotL2 + 1.)) * 0.5;
  float l = ( -1.0 + sqrt ( pbrInputs.alphaRoughness * (1.0 - NdotV2 ) / NdotV2 + 1.)) * 0.5;
  return (1.0 / max((1.0 + v + l ), 0.000001));
}
```

### Microfaced Distribution (D)

**Trowbridge-Reitz**
Implementation of microfaced distrubtion from [Average Irregularity Representation of a Roughened Surface for Ray Reflection](https://www.osapublishing.org/josa/abstract.cfm?uri=josa-65-5-531) by T. S. Trowbridge, and K. P. Reitz

```
float microfacetDistribution(PBRInfo pbrInputs)
{
    float roughnessSq = pbrInputs.alphaRoughness * pbrInputs.alphaRoughness;
    float f = (pbrInputs.NdotH * roughnessSq - pbrInputs.NdotH) * pbrInputs.NdotH + 1.0;
    return roughnessSq / (M_PI * f * f);
}
```

### Diffuse Term
The following equations are commonly used models of the diffuse term of the lighting equation.

以下方程式是光照方程中漫射项的常用模型。

**Lambert**
Implementation of diffuse from [Lambert's Photometria](https://archive.org/details/lambertsphotome00lambgoog) by Johann Heinrich Lambert

```
vec3 diffuse(PBRInfo pbrInputs)
{
    return pbrInputs.diffuseColor / M_PI;
}
```

**Disney**
Implementation of diffuse from [Physically-Based Shading at Disney](http://blog.selfshadow.com/publications/s2012-shading-course/burley/s2012_pbs_disney_brdf_notes_v3.pdf) by Brent Burley.  See Section 5.3.

```
vec3 diffuse(PBRInfo pbrInputs)
{
    float f90 = 2.0 * pbrInputs.LdotH * pbrInputs.LdotH * pbrInputs.alphaRoughness - 0.5;

    return (pbrInputs.diffuseColor / M_PI) * (1.0 + f90 * pow((1.0 - pbrInputs.NdotL), 5.0)) * (1.0 + f90 * pow((1.0 - pbrInputs.NdotV), 5.0));
}
```
