# WORK IN PROGRESS!

# The goal

Well, we are approaching the end of your short course of CG lectures. The goal for today is to compute shadows. **Attention, we are talking aboud hard shadows here, soft shadows computation is another story.**

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/07-shadows/50de2abe990efa345664f98c9464a4c8.png)

Up to this moment convex objects were shaded correctly by our simple local shading. Local means computation with light direction and the normal vector. Unfortunately, it does not produce correct results for non-convex objects. Here is the image we can got during previous lesson:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/07-shadows/b4af24130ecb1536703e4793308af425.png)

Why is there some light at the right (demon's right, our left) shoulder? Why do not we see a shadow from his left corn? Not good.

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

This shader simply copies the z-buffer into the framebuffer. Here is how it I call it from the main() function:

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

