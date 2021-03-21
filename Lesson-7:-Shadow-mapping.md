# The goal

Well, we are approaching the end of your short course of CG lectures. The goal for today is to compute shadows. **Attention, we are talking about hard shadows here, soft shadows computation is another story.** As always, the source code is [available](https://github.com/ssloy/tinyrenderer/tree/0c1d955e4f86c25f31f97e4f4563313ddba0c104).

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/07-shadows/50de2abe990efa345664f98c9464a4c8.png)

# Problem statement

Up to this moment convex objects were shaded correctly by our simple local shading. Local means computation with light direction and the normal vector. Unfortunately, it does not produce correct results for non-convex objects. Here is the image we can got during previous lesson:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/07-shadows/b4af24130ecb1536703e4793308af425.png)

Why is there some light at the right (demon's right, our left) shoulder? Why do not we see a shadow from his left horn? Not good.

The idea is really simple: we will do a two-pass rendering. First time we will render the image placing the camera at the light source position. It will allow to determine what parts are lit and what parts are hidden from the light. Then in the second pass we do a render taking in account the visibility information. Almost no difficulties here. Let us use this shader:

```C++
struct DepthShader : public IShader {
    mat<3,3,float> varying_tri;

    DepthShader() : varying_tri() {}

    virtual Vec4f vertex(int iface, int nthvert) {
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        gl_Vertex = Viewport*Projection*ModelView*gl_Vertex;          // transform it to screen coordinates
        varying_tri.set_col(nthvert, proj<3>(gl_Vertex/gl_Vertex[3]));
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        Vec3f p = varying_tri*bar;
        color = TGAColor(255, 255, 255)*(p.z/depth);
        return false;
    }
};
```

This shader simply copies the z-buffer into the framebuffer. Here is how I call it from the main() function:

```C++
     { // rendering the shadow buffer
        TGAImage depth(width, height, TGAImage::RGB);
        lookat(light_dir, center, up);
        viewport(width/8, height/8, width*3/4, height*3/4);
        projection(0);

        DepthShader depthshader;
        Vec4f screen_coords[3];
        for (int i=0; i<model->nfaces(); i++) {
            for (int j=0; j<3; j++) {
                screen_coords[j] = depthshader.vertex(i, j);
            }
            triangle(screen_coords, depthshader, depth, shadowbuffer);
        }
        depth.flip_vertically(); // to place the origin in the bottom left corner of the image
        depth.write_tga_file("depth.tga");
    }

    Matrix M = Viewport*Projection*ModelView;
```

I put the camera at the light source position (lookat(light_dir, center, up);) and then perform the render. Note that I keep the z-buffer, it is pointed by the **shadowbuffer** pointer. Also it is useful to note that in the last line I keep the object-to-screen transformation matrix. Here is the result of the shader's work:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/07-shadows/f743999b9d21aee9d0704c4036e18dce.png)

The second pass is naturally made with another shader:

```C++
struct Shader : public IShader {
    mat<4,4,float> uniform_M;   //  Projection*ModelView
    mat<4,4,float> uniform_MIT; // (Projection*ModelView).invert_transpose()
    mat<4,4,float> uniform_Mshadow; // transform framebuffer screen coordinates to shadowbuffer screen coordinates
    mat<2,3,float> varying_uv;  // triangle uv coordinates, written by the vertex shader, read by the fragment shader
    mat<3,3,float> varying_tri; // triangle coordinates before Viewport transform, written by VS, read by FS

    Shader(Matrix M, Matrix MIT, Matrix MS) : uniform_M(M), uniform_MIT(MIT), uniform_Mshadow(MS), varying_uv(), varying_tri() {}

    virtual Vec4f vertex(int iface, int nthvert) {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        Vec4f gl_Vertex = Viewport*Projection*ModelView*embed<4>(model->vert(iface, nthvert));
        varying_tri.set_col(nthvert, proj<3>(gl_Vertex/gl_Vertex[3]));
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        Vec4f sb_p = uniform_Mshadow*embed<4>(varying_tri*bar); // corresponding point in the shadow buffer
        sb_p = sb_p/sb_p[3];
        int idx = int(sb_p[0]) + int(sb_p[1])*width; // index in the shadowbuffer array
        float shadow = .3+.7*(shadowbuffer[idx]<sb_p[2]); 
        Vec2f uv = varying_uv*bar;                 // interpolate uv for the current pixel
        Vec3f n = proj<3>(uniform_MIT*embed<4>(model->normal(uv))).normalize(); // normal
        Vec3f l = proj<3>(uniform_M  *embed<4>(light_dir        )).normalize(); // light vector
        Vec3f r = (n*(n*l*2.f) - l).normalize();   // reflected light
        float spec = pow(std::max(r.z, 0.0f), model->specular(uv));
        float diff = std::max(0.f, n*l);
        TGAColor c = model->diffuse(uv);
        for (int i=0; i<3; i++) color[i] = std::min<float>(20 + c[i]*shadow*(1.2*diff + .6*spec), 255);
        return false;
    }
};
```

It is a copy of the [final shader](https://github.com/ssloy/tinyrenderer/wiki/Lesson-6:-Shaders-for-the-software-renderer#specular-mapping) from the previous lesson with one exception: I declared a constant matrix ```mat<4,4,float> uniform_Mshadow```, it allows me to transform screen coordinates of current fragment into screen coordinates inside the shadowbuffer! I'll explain how it is computed a bit later, let us see how I use it:

```C++
     Vec4f sb_p = uniform_Mshadow*embed<4>(varying_tri*bar); // corresponding point in the shadow buffer
        sb_p = sb_p/sb_p[3];
        int idx = int(sb_p[0]) + int(sb_p[1])*width; // index in the shadowbuffer array
        float shadow = .3+.7*(shadowbuffer[idx]<sb_p[2]);
```

varying_tri\*bar provides me screen coordinates of the pixel we currently draw; we augment it with 1 (recall the homogeneous coordinates stuff), then transform it with the magic matrix uniform_Mshadow and ta-da! We know xyz coordinates in the shadow buffer space. Now to determine whether the current pixel is lit or no it suffices to compare its z-coordinate with the value we stored in the shadow buffer.

Let me show you how I call the shader:

```C++
    Matrix M = Viewport*Projection*ModelView;

    { // rendering the frame buffer
        TGAImage frame(width, height, TGAImage::RGB);
        lookat(eye, center, up);
        viewport(width/8, height/8, width*3/4, height*3/4);
        projection(-1.f/(eye-center).norm());

        Shader shader(ModelView, (Projection*ModelView).invert_transpose(), M*(Viewport*Projection*ModelView).invert());
        Vec4f screen_coords[3];
        for (int i=0; i<model->nfaces(); i++) {
            for (int j=0; j<3; j++) {
                screen_coords[j] = shader.vertex(i, j);
            }
            triangle(screen_coords, shader, frame, zbuffer);
        }
        frame.flip_vertically(); // to place the origin in the bottom left corner of the image
        frame.write_tga_file("framebuffer.tga");
    }
```

Recall that the matrix M is the transformation matrix from the object space to the shadow buffer screen space. We return the camera back to its normal position, recompute the viewport matrix, the projection matrix and call the second shader.

We know that ```Viewport*Projection*ModelView``` transforms the object's coordinates into the (framebuffer) screen space. We need to know how to transform the framebuffer screen into the shadow screen. It is really simple: ```(Viewport*Projection*ModelView).invert()``` allows to convert framebuffer coordinates into object coordinates and then ```M*(Viewport*Projection*ModelView).invert()``` gives the transformation between the framebuffer and the shadow buffer.

All that is mighty well but there is a hiccup:
![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/07-shadows/164be1dce9e980d47a90159103b954a3.png)

Notice the ugly shadow rendering? This artifact is known as the [z-fighting](http://en.wikipedia.org/wiki/Z-fighting). Resolution of our buffers is insufficient to obtain precise results. How to solve the problem? I like brute force solutions:

```C++
        float shadow = .3+.7*(shadowbuffer[idx]<sb_p[2]+43.34); // magic coeff to avoid z-fighting
```

I simply move a bit one z-buffer with respect to another, it is sufficient to remove the artifact. Yes, it creates other problems (can you tell which ones?), but those are generally less visible. The final render is visible in the teaser image.

# Homework assignment

## shadow maps
I uploaded a Diablo model, try to add shadows to the render:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/07-shadows/e3cd704925f52b5466ab3c4f9fbab899.png)

## glow
Notice the crystal (and the eyes), they are glowing. How to achieve this effect?