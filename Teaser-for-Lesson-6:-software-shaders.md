# [WORK IN PROGRESS]

**Recall that all my source code here is meant to be compared with yours. Do not use my code, write your own. I am a bad programmer. Please, do the most insane shaders and send me images, I'll post them here.**

Time for fun. First of all, let us check the current state of the [source code](https://github.com/ssloy/tinyrenderer/tree/f037c7a0517a632c7391b35131f9746a8f8bb235):

* geometry.cpp+.h — 218 lines
* model.cpp+.h — 139 lines
* our_gl.cpp+.h — 102 lines
* main.cpp — 66 lines

For a total of 525 lines, exactly what we wanted. Please note that the only files responsible for actual rendering are in our_gl.* and main.cpp with **168 lines in total**.

![](https://hsto.org/getpro/habr/post_images/e3c/d70/492/e3cd704925f52b5466ab3c4f9fbab899.png)

# Refactoring the source code

Okay, our main.cpp is starting to grow too much, let us split it in two:
* our_gl.cpp+h - this part the programmer can not touch: roughly speaking, it is a binary of the OpenGL library
* main.cpp - here we can program all we want.

Now, what did I put into our_gl? ModelView, Viewport and Projection matrices along with initialization functions and the triangle rasterizer. That is all!

Here is the content of the file our_gl.h (I'll cover IShader structure later):

```C++
#include "tgaimage.h"
#include "geometry.h"

extern Matrix ModelView;
extern Matrix Viewport;
extern Matrix Projection;

void viewport(int x, int y, int w, int h);
void projection(float coeff=0.f); // coeff = -1/c
void lookat(Vec3f eye, Vec3f center, Vec3f up);

struct IShader {
    virtual ~IShader();
    virtual Vec3i vertex(int iface, int nthvert) = 0;
    virtual bool fragment(Vec3f bar, TGAColor &color) = 0;
};

void triangle(Vec4f *pts, IShader &shader, TGAImage &image, TGAImage &zbuffer);
```

File main.cpp now has 66 lines only, thus I give its complete listing (sorry for the long code, but I list it complete because I like it so much):

```C++
#include <vector>
#include <iostream>

#include "tgaimage.h"
#include "model.h"
#include "geometry.h"
#include "our_gl.h"

Model *model     = NULL;
const int width  = 800;
const int height = 800;

Vec3f light_dir(1,1,1);
Vec3f       eye(1,1,3);
Vec3f    center(0,0,0);
Vec3f        up(0,1,0);

struct GouraudShader : public IShader {
    Vec3f varying_intensity; // written by vertex shader, read by fragment shader

    virtual Vec4f vertex(int iface, int nthvert) {
        varying_intensity[nthvert] = std::max(0.f, model->normal(iface, nthvert)*light_dir); // get diffuse lighting intensity
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        return Viewport*Projection*ModelView*gl_Vertex; // transform it to screen coordinates
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        float intensity = varying_intensity*bar;   // interpolate intensity for the current pixel
        color = TGAColor(255, 255, 255)*intensity; // well duh
        return false;                              // no, we do not discard this pixel
    }
};

int main(int argc, char** argv) {
    if (2==argc) {
        model = new Model(argv[1]);
    } else {
        model = new Model("obj/african_head.obj");
    }

    lookat(eye, center, up);
    viewport(width/8, height/8, width*3/4, height*3/4);
    projection(-1.f/(eye-center).norm());
    light_dir.normalize();

    TGAImage image  (width, height, TGAImage::RGB);
    TGAImage zbuffer(width, height, TGAImage::GRAYSCALE);

    GouraudShader shader;
    for (int i=0; i<model->nfaces(); i++) {
        Vec4f screen_coords[3];
        for (int j=0; j<3; j++) {
            screen_coords[j] = shader.vertex(i, j);
        }
        triangle(screen_coords, shader, image, zbuffer);
    }

    image.  flip_vertically(); // to place the origin in the bottom left corner of the image
    zbuffer.flip_vertically();
    image.  write_tga_file("output.tga");
    zbuffer.write_tga_file("zbuffer.tga");

    delete model;
    return 0;
}
```

Let us see how it works. Skipping the headers, we declare few global constants: screen dimensions, camera position etc. I will explain the GouraudShader struct in the next paragraph, so let us skip it. Then it is the actual main() function:
* Parsing the .obj file
* Initialization of ModelView, Projection and Viewport matrices (recall that actual instances of these matrices are in the our_gl module)
* Iteration through all triangles of the model and rasterization of each triangle.

The last step is the most interesting. Outer loop iterates through all the triangles. Inner loop iterates through all the vertices of the current triangle and calls a vertex shader for each vertex.

**The main goal of the vertex shader is to transform the coordinates of the vertices. The secondary goal is to prepare data for the fragment shader.**

What happens after that? We call the rasterization routine. What happens inside the rasterizer we do not know (well, okay, we do know since we programmed it!) with one exception. We know that the rasterizer calls **our** routine for each pixel, namely, the fragment shader. Again, for each pixel inside the triangle the rasterizer calls our own callback, the fragment shader.

**The main goal of the fragment shader - is to determine the color of the current pixel. Secondary goal - we can discard current pixel by returning true.**

The rendering pipeline for the OpenGL 2 can be represented as follows (in fact, it is more or less the same for newer versions too):

![](http://3dgep.com/wp-content/uploads/2014/01/OpenGL-2.0-Programmable-Shader-Pipeline.png)

Because of the time limits I have for my course, I restrict myself to the OpenGL 2 pipeline and therefore to fragment and vertex shaders only. In newer versions of OpenGL there are other shaders, allowing, for example, to generate geometry on the fly. Well, in the above image all the stages we can not touch are shown in blue, whereas our callbacks are shown in orange. In fact, our main() function - is the **primitive processing** routine. It calls the vertex shader. We do not have primitive assembly here, since we are drawing dumb triangles only (in our code it is merged with the primitive processing). triangle() function - is the **rasterizer**, for each point inside the triangle it calls the **fragment shader**, then performs depth checks (z-buffer) and such.

That is all. You know what the shaders are and now you can create your own shaders.

Let us check the shader I listed above in the main.cpp. As the name shows, it is a Gouraud shader. Let me re-list the code:

```C++
    Vec3f varying_intensity; // written by vertex shader, read by fragment shader
    virtual Vec4f vertex(int iface, int nthvert) {
        varying_intensity[nthvert] = std::max(0.f, model->normal(iface, nthvert)*light_dir); // get diffuse lighting intensity
        Vec4f gl_Vertex = embed<4>(model->vert(iface, nthvert)); // read the vertex from .obj file
        return Viewport*Projection*ModelView*gl_Vertex; // transform it to screen coordinates
    }
```

vayring - is a reserved keyword in GLSL language, I have used varying_intensity as a name in order to show the correspondence (we will talk about GLSL in the [lesson 9](https://github.com/ssloy/tinyrenderer/wiki/Lesson-9:-Real-OpenGL-(GLSL)-application)). In varying variables we store data to be interpolated inside the triangle, and the fragment shaders gets the interpolated value (for the current pixel).

Let us re-list the fragment shader:

```C++
  Vec3f varying_intensity; // written by vertex shader, read by fragment shader
// [...]
    virtual bool fragment(Vec3f bar, TGAColor &color) {
        float intensity = varying_intensity*bar;   // interpolate intensity for the current pixel
        color = TGAColor(255, 255, 255)*intensity; // well duh
        return false;                              // no, we do not discard this pixel
    }
```

This routine is called for each pixel inside the triangle we draw; as an input it receives [barycentric coordinates](https://en.wikipedia.org/wiki/Barycentric_coordinate_system) for interpolation of varying_ data.



![](http://www.loria.fr/~sokolovd/infographie/04-geometry/tmp/african_head_nm_tangent.png)