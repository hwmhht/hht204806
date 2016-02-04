# Last necessary bit of geometry

Today we are finishing with the part that i like a lot, but many readers find boring. Once you have mastered today's material, you can move to the next lesson and there we will actually do renders. To brighten you up here is the head we already know rendered using [Gouraud shading](https://en.wikipedia.org/wiki/Gouraud_shading):

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/bcdf0bba53495b4ebc86ba45f03d255e.png)

I removed all the textures. Gouraud shading is really simple. Our kind 3d artists gave us normal vectors to each vertex of the model, they can be found in "vn x y z" lines of the .obj file. We compute the intensity per vertex (and not per triangle as before for the flat shading) and then simply interpolate the intensity inside each triangle as we already did for z or uv coordinates. 

By the way, in the case when 3d artists are not so kind, you can recompute the normal vectors as an average of normals to all facets incident to the vertex. Current code i used to generate this image can be found [here](https://github.com/ssloy/tinyrenderer/tree/10723326bb631d081948e5346d2a64a0dd738557).

# Change of basis in 3D space

In Euclidean space coordinates can be given by a point (the origin) and a basis. What does it mean that point P has coordinates (x,y,z) in the frame (O, i,j,k)? It means that the vector OP can be expressed as follows:

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f00.svg)

Now image that we have another frame (O', i',j',k'). How do we transform coordinates given in one frame to another? First of all let us note that since (i,j,k) and (i',j',k') are bases of 3D, there exists a (non degenerate) matrix M such that:

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f01.svg)

Let us draw an illustration:

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f66a0139058ab1d1025dbfd8cd401389.png)

Then let us re-express the vector OP:

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f02.svg)

Now let us substitute (i',j',k') in the right part with the change of basis matrix:

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f03.svg)

And it gives us the formula to transform coordinates from one frame to another:

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f04.svg)


# Let us create our own gluLookAt

OpenGL and, as a consequence, our tiny renderer are able to draw scenes **only with the camera located on the z-axis.** If we want to move the camera, no problem, we can move all the scene, leaving the camera immobile.

Let us put the problem this way: we want to draw a scene with a camera situated in point **e** (eye), the camera should be pointed to the point **c** (center) in such way that a given vector **u** (up) is to be vertical in the final render.

Here is an illustration:

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/b94dd4a591514fd66a91a6e4cc065644.png)

It means that we want to do the rendering in the frame (c, x',y',z'). But then our model is given in the frame (O, x,y,z)... No problem, all we need is to compute the transformation of the coordinates. Here is a C++ code computing the necessary 4x4 matrix ModelView:

```C++
void lookat(Vec3f eye, Vec3f center, Vec3f up) {
    Vec3f z = (eye-center).normalize();
    Vec3f x = cross(up,z).normalize();
    Vec3f y = cross(z,x).normalize();
    Matrix Minv = Matrix::identity();
    Matrix Tr   = Matrix::identity();
    for (int i=0; i<3; i++) {
        Minv[0][i] = x[i];
        Minv[1][i] = y[i];
        Minv[2][i] = z[i];
        Tr[i][3] = -center[i];
    }
    ModelView = Minv*Tr;
}
```

Note that z' is given by the vector **ce** (do not forget to normalize it, it helps later). How do we compute x'? Simply by a cross product between **u** and **z'**. Then we compute y', such that it is orthogonal to already calculated x' and z' (let me remind you that in our problem settings **ce** and **u** are not necessarily orthogonal). The very last step is a translation of the origin to the center **c** and our transformation matrix is ready. Now it suffices to get any point with coordinates (x,y,z,1) in the model frame, multiply it by the matrix ModelView and we get the coordinates in the camera frame! By the way, the name ModelView comes from OpenGL terminology.

# Viewport

If you followed this course from the beginning, you should remember strange lines like this one:
```C++
screen_coords[j] = Vec2i((v.x+1.)*width/2., (v.y+1.)*height/2.);
```

What does it mean? It means that i have a point Vec2f v, it belongs to the square [-1,1]*[-1,1]. I want to draw it in the image of (width, height) dimensions. Value (v.x+1) is varying between 0 and 2, (v.x+1)/2 between 0 and 1, and (v.x+1)*width/2 sweeps all the image. Thus we effectively mapped the bi-unit square onto the image.

But now we are getting rid of these ugly constructs, and i want to rewrite all the computiations in the matrix form. Let us consider the following C++ code:

```C++
Matrix viewport(int x, int y, int w, int h) {
    Matrix m = Matrix::identity(4);
    m[0][3] = x+w/2.f;
    m[1][3] = y+h/2.f;
    m[2][3] = depth/2.f;

    m[0][0] = w/2.f;
    m[1][1] = h/2.f;
    m[2][2] = depth/2.f;
    return m;
}
```

This code creates this matrix:

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f08.svg)

It means that the bi-unit cube [-1,1]*[-1,1]*[-1,1] is mapped onto the screen cube [x,x+w]*[y,y+h]*[0,d]. Right, cube, and not a rectangle, this is because of the depth computations with the z-buffer. Here d is the resolution of the z-buffer. I like to have it equal to 255 because of simplicity of dumping black-and-white images of the z-buffer for debugging.

In the OpenGL terminology this matrix is called viewport matrix.

# Chain of coordinate transformations

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f05.svg)

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f06.svg)

![](http://www.loria.fr/~sokolovd/cg-course/05-camera/img/f07.svg)
