---
layout: page
title: vxgi 
description: Real Time Global Illumination Using Voxel Cone Tracing 
img: assets/img/vxgi2.png
toc:
  sidebar: left
importance: 1
---
{% include figure.liquid
    path="assets/img/vxgi-emissive.png"
    width="500px"
    loading="eager"
    caption="
        Emissivity at 30fps on Integrated Intel Graphics
    "
    class="mx-auto d-block rounded z-depth-1"
%}

Global illumination (GI) adds realism to computer graphics by simulating how light interacts with surfaces. Traditional raytracing methods for GI are computationally expensive, leading to slower performance. To address this, Voxel Cone Tracing (VCT) offers a balance between performance and visual quality by approximating indirect lighting through cone tracing. Below, I detail the pipeline and techniques used in my project.

## The VXGI Pipeline

Voxel Cone Tracing operates on a hybrid rendering pipeline:
- **Voxelization** is achieved using GPU hardware rasterization.
- **Primary rays** are rasterized for direct lighting.
- **Secondary rays** are cone-traced to compute indirect global illumination.

## Voxelization Process

Voxelization is central to this technique, converting 3D geometry into a voxel representation for lighting computations. The process includes:

### Rasterization:
- The GPU generates voxels, with thin-surface voxelization ensuring accuracy.
- Conservative rasterization ensures fragments are created for all pixels intersecting a primitive using NVIDIA’s `GL_NV_conservative_raster` extension.

### Dominant Axis Projection:
- Triangles are projected along their dominant axis for efficient voxelization.

{% include figure.liquid
    path="assets/img/vxgi-das.png"
    width="250px"
    loading="eager"
    caption="
        Projection along all three axes
    "
    class="mx-auto d-block rounded z-depth-1"
%}

$$l_{\{x,y,z\}} = |n · v_{\{x,y,z\}} |$$

- Inside the Geometry Shader

```glsl
// select the dominant axis
float axis = max(geometryNormal.x, max(geometryNormal.y, geometryNormal.z));

if (axis == geometryNormal.x) {
    gl_Position = vec4(outputPosition.zy, 1.0, 1.0);
} else if (axis == geometryNormal.y) {
    gl_Position = vec4(outputPosition.xz, 1.0, 1.0);
} else if (axis == geometryNormal.z) {
    gl_Position = vec4(outputPosition.xy, 1.0, 1.0);
}
EmitVertex();
```

### Direct Light Injection and Mipmapping:
- Direct lighting is computed and stored in a 3D image using OpenGL's `imageStore()`.
- Automatic mipmap generation with `glGenerateMipmap()` handles multi-resolution sampling.

## Cone Tracing for Global Illumination

Cone tracing approximates the rendering equation for indirect lighting:

$$ L_o(\mathbf{p}, \omega_o) = L_e(\mathbf{p}, \omega_o) + \int_{\Omega} f(\mathbf{p}, \omega_i, \omega_o) \cdot L_i(\mathbf{p}, \omega_i) \cdot (\mathbf{n} \cdot \omega_i) \, d\omega_i $$

This integral is replaced by a summation of N cones.

{% include figure.liquid
    path="assets/img/vxgi-ao.png"
    width="250px"
    loading="eager"
    caption="
        Ambient Occlusion
    "
    class="mx-auto d-block rounded z-depth-1"
%}

$$
L_o(\mathbf{x}, \omega) \approx L_e(\mathbf{x}, \omega) + \frac{1}{N} \sum_{i=1}^{N} V_c(\mathbf{x}, \omega_i)
$$

Each cone is raymarched through the voxelized 3D mipmap.

{% include figure.liquid
    path="assets/img/vxgi-cone.svg"
    width="350px"
    loading="eager"
    caption="
        Raymarching along the normal
    "
    class="mx-auto d-block rounded z-depth-1"
%}

We define the diameter d as:

$$d = 2t \times \tan(\frac{\theta}{2})$$

Sampling uses mipmap levels determined by:

$$ V_{\text{level}} = \log_2(\frac{d}{V_{\text{size}}}) $$

Lighting values are accumulated volumetrically using alpha blending:

$$C = \alpha C + (1 - \alpha)\alpha_2C_2$$

$$\alpha = \alpha + (1 - \alpha)\alpha_2$$

### Indirect Light Components

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/vxgi-diffuse.png" class="img-fluid rounded z-depth-1" %}
        <div class="caption">Indirect Diffuse</div>
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/vxgi-specular.png" class="img-fluid rounded z-depth-1" %}
        <div class="caption">Indirect Specular</div>
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid loading="eager" path="assets/img/vxgi-brdf.png" class="img-fluid rounded z-depth-1" %}
        <div class="caption">Complete GI</div>
    </div>
</div>

## Results

{% include figure.liquid
    path="assets/img/vxgi-result.png"
    width="500px"
    loading="eager"
    caption="
        Bomb Rush 
    "
    class="mx-auto d-block rounded z-depth-1"
%}

{% include figure.liquid
    path="assets/img/vxgi-crytek.png"
    width="500px"
    loading="eager"
    caption="
        Crytek Sponza
    "
    class="mx-auto d-block rounded z-depth-1"
%}
