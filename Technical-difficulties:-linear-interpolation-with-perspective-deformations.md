# Motivation

![](https://hsto.org/getpro/habr/post_images/4cb/4db/4fa/4cb4db4fab48cd386dd7c543ed7df10d.png)

Okay, let us sum up how real-time rendering works. We have a triangle in world coordinates, it is projected to the screen. Then for each pixel inside the triangle we compute its barycentric coordinates with respect to the triangle, pass the barycentric coordinates to the shader and then the shader can interpolate anything inside the triangle: texture coordinates, normals, colors, whatever.

[Here is the commit](https://github.com/ssloy/tinyrenderer/tree/8294312644c7ff103adcc4b2f5b068cba393498e) that works exactly in this manner. And its result is shown in the left image of the above figure. [Here is the correction](https://github.com/ssloy/tinyrenderer/commit/0c8afb6d8350de46518e0539120662af962ba46f) with correct perspective-aware interpolation, check the floor tilings. All I did was to pass to the shaders bc_clip coordinates and not bc_screen. Hm. Let us find why.

# Non-linearity

The main problem of is non-linearity of the chain of transformations we have. To pass from homogeneous coordinates to 3D we divide by the last component, breaking the linearity of transformations. Therefore, we do not have the right to use screen-space barycentric coordinates to interpolate anything in the original space (before the perspective transformation).

Let us formulate our problem in the following way. We know that some point P belonging to a triangle ABC is transformed to point P' after the perspective division:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/td-perspective-correction/f04.png)

We know barycentric coordinates of the point P' with respect to the triangle A'B'C' (screen-space coordinates of the triangle):

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/td-perspective-correction/f07.png)

Then, knowing screen-space coordinates A'B'C' and barycentric coordinates of P' with respect to A'B'C', we need to find barycentric coordinates of P with respect to the original triangle ABC:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/td-perspective-correction/f06.png)

No problem, let us express P':

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/td-perspective-correction/f08.png)

Multiply it by rP.z+1:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/td-perspective-correction/f09.png)

So we obtain the expression P = [ABC]*[some vector]. But it is the definition of barycentric coordinates! We are really close to the goal. What do we know and what is unknown?

Alpha-beta-gamma-all-prime are known. rA.z+1, rB.z+1, rC.z+1 are known too, those are the coordinates of the triangle passed to the rasterizer routine. We need to find the last thing, namely, rP.z+1, or the z-coordinate of the point P. With its aid we can find the coordinates of the point P. Wow, an ugly loop...

In fact, it is not a problem, let us break the loop. In (normalized) barycentric coordinates the sum of all components is equal to 1, so alpha+beta+gamma=1:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/td-perspective-correction/f10.png)

Now all the symbols are known and we can transform screen-space barycentric coordinates to the world-space ones. The transformation is not linear, but it allows to compensate the non-linearity of the perspective distortion, resulting in linear interpolation in the world-space coordinates. Check the right image of the top figure.
