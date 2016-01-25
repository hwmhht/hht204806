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

# Timings: fourth attempt

_**Warning**: compiler’s optimizator (g++ -O3) is often better than you (and me) in creating a fast code. This section is here for historical/cultural reasons._

This code works great. That’s exactly the kind of complexity I want to see in the final version or our renderer. It is definitely inefficient (multiple divisions, and the like), but it is short and readable. Note that it has no asserts and no checks on going beyond the borders, which is bad. In these articles I try not to overload this particular code, as it gots read a lot. At the same time, I systematically remind of the necessity to perform checks.

So, the previous code works fine, but we can optimize it. Optimization is a dangerous thing. We should be clear about the platform the code will run on. Optimizing the code for a graphics card or just for a CPU — are completely different things. Before and during any optimization, the code must be profiled. Try to guess, which operation is the most recourse-intensive operation here?

For tests, 1,000,000 times I draw 3 line segments we have drawn before. My CPU is Intel®; Core(TM) i5-3450 CPU @ 3.10GHz. For each pixel, this code calls the TGAColor copy constructor. Which is 1000000 * 3 line segments * approximately 50 pixels per line segment. Quite a lot of calls, isn’t it? Where to start with optimization? The profiler will tell us.

I compiled the code with g++ -ggdb -g3 -pg -O0 keys, and then ran gprof:

``` 
%   cumulative   self              self     total 
 time   seconds   seconds    calls  ms/call  ms/call  name 
 69.16      2.95     2.95  3000000     0.00     0.00  line(int, int, int, int, TGAImage&, TGAColor) 
 19.46      3.78     0.83 204000000     0.00     0.00  TGAImage::set(int, int, TGAColor) 
  8.91      4.16     0.38 207000000     0.00     0.00  TGAColor::TGAColor(TGAColor const&) 
  1.64      4.23     0.07        2    35.04    35.04  TGAColor::TGAColor(unsigned char, unsigned char, unsigned char, unsigned char) 
  0.94      4.27     0.04                             TGAImage::get(int, int)
```

10% of the time are spent on copying the color. But then 70% are performed in calling line()! That’s where we will optimize.

# Fourth attempt contiued

We should note that each division has the same divisor. Let’s take it out of the loop. The error variable gives is the distance to the best straight line from our current (x, y) pixel. Each time error is greater than one pixel, we increase (or decrease) y by one, and decrease the error by one as well.

The code is available [here](https://github.com/ssloy/tinyrenderer/tree/2086cc7c082f4aec536661d7b4ab8a469eb0ce06).
 
```C++
void line(int x0, int y0, int x1, int y1, TGAImage &image, TGAColor color) { 
    bool steep = false; 
    if (std::abs(x0-x1)<std::abs(y0-y1)) { 
        std::swap(x0, y0); 
        std::swap(x1, y1); 
        steep = true; 
    } 
    if (x0>x1) { 
        std::swap(x0, x1); 
        std::swap(y0, y1); 
    } 
    int dx = x1-x0; 
    int dy = y1-y0; 
    float derror = std::abs(dy/float(dx)); 
    float error = 0; 
    int y = y0; 
    for (int x=x0; x<=x1; x++) { 
        if (steep) { 
            image.set(y, x, color); 
        } else { 
            image.set(x, y, color); 
        } 
        error += derror; 
        if (error>.5) { 
            y += (y1>y0?1:-1); 
            error -= 1.; 
        } 
    } 
} 
```

Here is the output of gprof:

```
%   cumulative   self              self     total 
 time   seconds   seconds    calls  ms/call  ms/call  name 
 38.79      0.93     0.93  3000000     0.00     0.00  line(int, int, int, int, TGAImage&, TGAColor) 
 37.54      1.83     0.90 204000000     0.00     0.00  TGAImage::set(int, int, TGAColor) 
 19.60      2.30     0.47 204000000     0.00     0.00  TGAColor::TGAColor(int, int) 
  2.09      2.35     0.05        2    25.03    25.03  TGAColor::TGAColor(unsigned char, unsigned char, unsigned char, unsigned char) 
  1.25      2.38     0.03                             TGAImage::get(int, int) 
```
