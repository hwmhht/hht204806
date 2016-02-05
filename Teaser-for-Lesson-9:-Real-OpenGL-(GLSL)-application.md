# The goal

Once upon a time I had to render (fast) molecules. For example, a molecule can be represented as a set of spheres, like in the following image:

![](https://hsto.org/getpro/habr/post_images/ca2/e9b/7a2/ca2e9b7a235690715acd5dc35da4d919.png)

This is a virus consisting of roughly three million atoms. You can download the model from a wonderful site called [rcsb.org](http://www.rcsb.org/pdb/explore.do?structureId=2BTV). 

It is a nice task to start learning shaders. First I will show how to create an OpenGL context and how to link our shaders.

# OpenGL helloworld

As usual, I created a [repo](https://github.com/ssloy/glsltuto/tree/006d7a1be29e2513af6700db7ed0d0063e859a2e). *Attention, this is a separate repository from the tinyrenderer, since it actually uses 3rd party libraries.* At the moment OpenGL does not provide a cross-platform way to create the rendering context. Here I use libraries GLUT, GLU and GLEW. 

**Attention,** *glut is really outdated, my choice is based on the fact that in our university hardware as well as software are really old. I do not even have a C++11 compiler. You should probably use sfml/sdl other. I am not worried though about glut, since this writing focuses on shaders (GPU side) and not on OpenGL context (CPU side) *

[Here](https://github.com/ssloy/glsltuto/blob/006d7a1be29e2513af6700db7ed0d0063e859a2e/main.cpp) is a code to draw [the Utah teapot](http://en.wikipedia.org/wiki/Utah_teapot).

