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