# Permanently under construction

This page is not meant to be a FAQ section; instead I show here several examples of bugs me and my students were able to fix *without* looking into the source code, solely by observing things in the images.


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


