# First attempt
The goal of the first lesson is to render the wire mesh. To do this, we should learn how to draw line segments. We can simply read what Bresenham’s line algorithm is, but let’s write code ourselves. How does the simplest code that draws a line segment between (x0, y0) and (x1, y1) points look like? Apparently, something like this :

```C++ 
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    for (float t=0.; t<1.; t+=.01) { 
        int x = x0*(1.-t) + x1*t; 
        int y = y0*(1.-t) + y1*t; 
        image.set(x, y, color); 
    } 
}
```

![](http://www.loria.fr/~sokolovd/cg-course/01-bresenham/img/c3c2ea8819.png)

The snapshot of the code is available [here](https://github.com/ssloy/tinyrenderer/tree/d0703acf18c48f2f7d00e552697d4797e0669ade).

# Second attempt

The problem with this code (in addition to efficiency) is the choice of the constant, which I took equal to .01. If we take it equal to .1, our line segment will look like this:

![](http://www.loria.fr/~sokolovd/cg-course/01-bresenham/img/62a16a5321.png)￼

We can easily find the necessary step: it’s just the number of pixels to be drawn. The simplest (with errors!) code looks something like the following:

```C++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    for (int x=x0; x<=x1; x++) { 
        float t = (x-x0)/(float)(x1-x0); 
        int y = y0*(1.-t) + y1*t; 
        image.set(x, y, color); 
    } 
}
```

Caution! The first source of errors in such code of my students is the integer division, like (x-x0)/(x1-x0). Then, if we try to draw the following lines with this code:
 
```C++
line(13, 20, 80, 40, image, white); 
line(20, 13, 40, 80, image, red); 
line(80, 40, 13, 20, image, red);
```


![](http://www.loria.fr/~sokolovd/cg-course/01-bresenham/img/097a691f9e.png)

It turns out that one line is good, the second one is with holes, and there’s no third line at all. Note that the first and the second lines (in the code) give the same line of different colors. We have already seen the white one, it is drawn well. I was hoping to change the color of the white line to red, but could not do it. It’s a test for symmetry: the result of drawing a line segment should not depend on the order of points: the (a,b) line segment should be exactly the same as the (b,a) line segment.

# Third attempt

There are holes in one of the line segments due to the fact that its height is greater than the width. My students often suggest the following fix:

```C++
if (dx>dy) {for (int x)} else {for (int y)}
```

Holy cow!

```C++ 
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { // if the line is steep, we transpose the image 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { // make it left−to−right 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    for (int x=x0; x<=x1; x++) { 
        float t = (x-x0)/(float)(x1-x0); 
        int y = y0*(1.-t) + y1*t; 
        if (steep) { 
            image.set(y, x, color); // if transposed, de−transpose 
        } else { 
            image.set(x, y, color); 
        } 
    } 
}
```

![](http://www.loria.fr/~sokolovd/cg-course/01-bresenham/img/3e8e5c7d26.png)
