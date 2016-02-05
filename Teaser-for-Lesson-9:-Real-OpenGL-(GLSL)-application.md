# The goal

Once upon a time I had to render (fast) molecules. For example, a molecule can be represented as a set of spheres, like in the following image:

![](https://hsto.org/getpro/habr/post_images/ca2/e9b/7a2/ca2e9b7a235690715acd5dc35da4d919.png)

This is a virus consisting of roughly three million atoms. You can download the model from a wonderful site called [rcsb.org](http://www.rcsb.org/pdb/explore.do?structureId=2BTV). 

It is a nice task to start learning shaders. First I will show how to create an OpenGL context and how to link our shaders.

# OpenGL helloworld

As usual, I created a [repo](https://github.com/ssloy/glsltuto/tree/006d7a1be29e2513af6700db7ed0d0063e859a2e). *Attention, this is a separate repository from the tinyrenderer, since it actually uses 3rd party libraries.* At the moment OpenGL does not provide a cross-platform way to create the rendering context. Here I use libraries GLUT, GLU and GLEW. 

**Attention,** *glut is really outdated, my choice is based on the fact that in our university hardware as well as software are really old. I do not even have a C++11 compiler. You should probably use sfml/sdl other. I am not worried though about glut, since this writing focuses on shaders (GPU side) and not on OpenGL context (CPU side) *

[Here](https://github.com/ssloy/glsltuto/blob/006d7a1be29e2513af6700db7ed0d0063e859a2e/main.cpp) is a code to draw [the Utah teapot](http://en.wikipedia.org/wiki/Utah_teapot).

Let us see how it works starting from main() function:
* First line initializes the library, telling that we will use two framebuffers, colors and a z-buffer
* Then we specify window dimensions, position, title and a background color, here it is blue
* Next something interesting happens: glutDisplayFunc, glutReshapeFunc and glutKeyboardFunc specify our callback functions that will be called on redraw, redimension and keyboard press events.
* Next a bit of checkboxes telling us that yes, we actually use z-buffer, we have a bit of lighting etc
* Finally - the call to the main window loop. glutMainLoop is working while the operating system shows the window.

Here keyboard processing is simple, I quit the application (a bit brutally) after hitting the ESC key. On resize event I pass to OpenGL new window size and specify that our projection is still orthogonal and we must map the bi-unit square [-1,1]x[-1,1] onto the screen.

The most interesting part is inside our render_scene() callback:
* first we clear our frame- and z-buffers
* next we clear our ModelView matrix and load inside the camera position (here it is constant)
* set the red color
* draw the teapot
* swap the framebuffers

Finally we should get the following picture:
![](https://habrastorage.org/getpro/habr/post_images/07a/035/9e5/07a0359e5c30889f63555cd4efa85624.png)

# GLSL helloworld

[Here](https://github.com/ssloy/glsltuto/tree/ebc9594a594bcedd7e91a5880bfef8e25ba81044) you can find source code of the simplest use of shaders. Github is really handy to see the changes between versions, compare what was [actually changed](https://github.com/ssloy/glsltuto/commit/ebc9594a594bcedd7e91a5880bfef8e25ba81044).

The image we should get is the following one:
[](https://habrastorage.org/getpro/habr/post_images/ec8/be3/fe5/ec8be3fe50ec9258ad9bd5bb328c4c8e.png)

What were the changes? First, I added two new files: frag_shader.glsl and vert_shader.glsl, written not in C++, but in [GLSL](http://en.wikipedia.org/wiki/OpenGL_Shading_Language). Those are our shaders, that we feed to the GPU. In main.cpp we added necessary wrapping code telling the GPU to use these shaders.

More precisely, I created prog_hdlr handler and linked the source code of shaders: first we read the text files, then we compile it on-the-fly and then link to the program handler.

# Drawing a "molecule" with standard OpenGL means