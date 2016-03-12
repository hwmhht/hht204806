# Permanently under construction

This page is not meant to be a FAQ section; instead I show here several examples of bugs me and my students were able to fix *without* looking into the source code, solely by observing things in the images. No made up situations, real student pain is shown. The description of bugs is kept to a bare minimum, these images speak for themselves.

# Triangle rasterization: line sweeping

Frequent bug: while sorting by y-coordinate, the vertices are sorted, but the data coming with the vertices is not.

### Gouraud shading, forgot to sort intensities
![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/line_sweeping/flat_shading_bad_sort.png)

### Z-buffer, forgot to sort z coordinates

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/line_sweeping/zbuffer_bad_sort.jpg)

### Horizontal sort of two vertices splitting the triangle

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/line_sweeping/bad_split_flip.jpg)


### Here are two examples of rounding errors creating holes in/between triangles:

Bad y interpolation while sweeping the line:
![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/line_sweeping/rounding/interpolated_y_coordinate.png)

The vertices were not casted to int prior to filling:
![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/line_sweeping/rounding/double_vertices_wold_be_better_to_cast_to_int.png)

# Bad data parsing

Forgot to decrement indices of vertices (remember in wavefront obj indices are starting from 1, not 0)

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/parsing/obj_decrement.png)


Few broken reads of texture coordinates, notice that all the fan of triangles incident to a vertex is messy, thus the vertex is broken:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/parsing/corrupted_vt_read.jpg)


All "v", "vt" and "vn" lines are read in the same array, thus effectively killing any sense in the model. Very roundish shape of the rendering gives the idea about "vn" confused with "v". Prior to the bugfix and after:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/parsing/vt_vn_interpreted_as_v_as_well.jpg)


# Specular map

Frequent bug, anything elevated to the zero power equals to one:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/specular/power0.jpg)

# Texturing

This one needs a circular permutation of barycentric coordinates in the uv-interpolation:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/uv/barycentric_coordinates_circular_permutation.jpg)

The texture needs to be vertically flipped:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/uv/texture_flip.jpg)

While visually ressembling to the above one, this one is completely different (why?). Here the student used xy pixel coordinates (instead of uv) to fetch a color from the texture :

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/a7c5da19378566533bd780918fc66323226467cf/troubleshooting/uv/xy_and_not_uv_read_from_texture.jpg)

# Lighting

Light direction (instead of view direction) was used for backface culling. Prior to and after bugfix.

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/troubleshooting/light/bad_backface_culling.png)


Broken normal map reading, before and after. [0,255]^3 RGB were brought to XYZ living in [0,2]^3 and not [-1,1]^3.

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/troubleshooting/light/broken_normal_map.jpg)


Light and/or normal vectors were not normalized in computation of flat shading, resulting in overflowing unsigned char. Before/after images.

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/troubleshooting/light/not_normalized.png)

Same bug with textured model:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/troubleshooting/light/not_normalized2.jpg)


A bug correlated to above: trying to assign negative colors overflows unsigned chars. Dot product of two normalized vectors varies between -1 and 1. Here on the right fabs() of the intensity is shown, simple clamp at zero would produce right image of the above pair.

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/troubleshooting/light/negative_colors.png)

# Bad camera

Negative focal length, clearly a bad camera coefficient c was used in <a href="https://github.com/ssloy/tinyrenderer/wiki/Lesson-4:-Perspective-projection#let-us-sum-up-the-main-formula-for-today">formula (2)</a>

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/961cc846891d6d978e45414a2da6fc75a2c59036/troubleshooting/bad_camera.jpg)

# Voxels

Quote from [here](https://www.reddit.com/r/VoxelGameDev/comments/465olm/i_accidentally_made_some_voxels_while_working_on/):

_The code to render triangles to the screen takes a lot of computation time. While I was experimenting with more efficient methods, I commented out the code that calculates the barycentric coordinates, basically it finds out where to plot pixels of the triangle in the bounding box. It was replaced with a literal vector as a dummy output. As a result, all the pixels of the box are rendered, producing the voxel look. Given that it's a 3D renderer they are all still depth sorted and texture-colored. The last two pictures show the model with a SSAO post-effect using the depth buffer as a reference._


![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/troubleshooting/diablo_voxelized.png)