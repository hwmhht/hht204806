# Filling triangles

Hi, everyone. It’s me.

![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/img/cfa0f3a9d9.png)
￼
More precisely, it is a model of my face rendered in the program we will create in the next hour or two. Last time we drew the wire mesh of a three-dimensional model. This time, we will fill polygons, or rather triangles. In fact, OpenGL triangulates almost any polygon, so there’s no need to consider sophisticated cases.

_Let me remind you, this series of articles is designed to allow you to program yourself. When i said that in two hours you can draw a picture like the one above, i do not mean the time to read my code. It’s time for creating your code from scratch. My code is provided here purely to compare your (working) program with mine. I am an awful programmer, it is very likely that you are a better one. Do not simply copy-paste my code. Any comments and questions are welcome._

## Old-school method: line sweeping

Thus, the task is to draw two-dimensional triangles. For motivated students it normally takes a couple of hours, even if they are bad programmers. Last time we saw Bresenham’s line drawing algorithm. Today’s task is to draw a filled triangle. Funny enough, but this task is not trivial. I don’t know why, but I know that it’s true. Most of my students struggle with this simple task.
So, the initial stub will look like this:
 
```C++
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    line(t0, t1, image, color); 
    line(t1, t2, image, color); 
    line(t2, t0, image, color); 
} 
[...] 
    Vec2i t0[3] = {Vec2i(10, 70),   Vec2i(50, 160),  Vec2i(70, 80)}; 
    Vec2i t1[3] = {Vec2i(180, 50),  Vec2i(150, 1),   Vec2i(70, 180)}; 
    Vec2i t2[3] = {Vec2i(180, 150), Vec2i(120, 160), Vec2i(130, 180)}; 
    triangle(t0[0], t0[1], t0[2], image, red); 
    triangle(t1[0], t1[1], t1[2], image, white); 
    triangle(t2[0], t2[1], t2[2], image, green);
```

![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/img/41060d3251.png)

As usual, the appropriate commit is available on [github](https://github.com/ssloy/tinyrenderer/tree/7e46cc57fa3f5a41129d6b6fefe4e77f77b8aa84). The code is simple: I provide three triangles for the initial debugging of your code. If we invoke line() inside the triangle function, we’ll get the contour of the triangle. How to draw a filled triangle?

A good method of drawing a triangle must have the following features:

* It should be (surprise!) simple and fast.
* It should be symmetrical: the picture should not depend on the order of vertices passed to the drawing function.
* If two triangles have two common vertices, there should be no holes between them because of rasterization rounding.
* We could add more requirements, but let’s do with these ones. Traditionally a line sweeping is used:

1. Sort vertices of the triangle by their y-coordinates;
2. Rasterize simultaneously the left and the right sides of the triangle;
3. Draw a horizontal line segment between the left and the right boundary points.

At this point my students start to loose the firm ground: which segment is the left one, which one is right? Besides, there are three segments in a triangle... Usually, after this introduction I leave my students for about an hour: once again, reading my code is much less valuable than comparing your own code with mine.

[One hour passes]

How do I draw a triangle? Once again, if you have a better method, I’d be glad to adopt it. Let us assume that we have three points of the triangle: t0, t1, t2, they are sorted in ascending order by the y-coordinate. Then, the boundary A is between t0 and t2, boundary B is between t0 and t1, and then between t1 and t2.
 
```C++
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!) 
    if (t0.y>t1.y) std::swap(t0, t1); 
    if (t0.y>t2.y) std::swap(t0, t2); 
    if (t1.y>t2.y) std::swap(t1, t2); 
    line(t0, t1, image, green); 
    line(t1, t2, image, green); 
    line(t2, t0, image, red); 
}
```

Here boundary A is red, and boundary B is green.

![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/img/3a5643f513.png)￼

Unfortunately, boundary B is made of two parts. Let us draw the bottom half of the triangle by cutting it horizontally:

```C++
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!) 
    if (t0.y>t1.y) std::swap(t0, t1); 
    if (t0.y>t2.y) std::swap(t0, t2); 
    if (t1.y>t2.y) std::swap(t1, t2); 
    int total_height = t2.y-t0.y; 
    for (int y=t0.y; y<=t1.y; y++) { 
        int segment_height = t1.y-t0.y+1; 
        float alpha = (float)(y-t0.y)/total_height; 
        float beta  = (float)(y-t0.y)/segment_height; // be careful with divisions by zero 
        Vec2i A = t0 + (t2-t0)*alpha; 
        Vec2i B = t0 + (t1-t0)*beta; 
        image.set(A.x, y, red); 
        image.set(B.x, y, green); 
    } 
}
```

![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/img/d8e0575a00.png)

Note that the segments are not continuous. Last time when we drew straight lines we struggled to get continuous segments and here i did not bother with rotating the image (remember the xy swapping?). Why? We fill the triangles aftewards, that’s why. If we connect the corresponding pairs of points by horizontal lines, the gaps disappear:

![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/img/c1f95127ad.png)

Now, let us draw the second (upper) half of the triangle. We can do this by adding a second loop:
 
```C++
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!) 
    if (t0.y>t1.y) std::swap(t0, t1); 
    if (t0.y>t2.y) std::swap(t0, t2); 
    if (t1.y>t2.y) std::swap(t1, t2); 
    int total_height = t2.y-t0.y; 
    for (int y=t0.y; y<=t1.y; y++) { 
        int segment_height = t1.y-t0.y+1; 
        float alpha = (float)(y-t0.y)/total_height; 
        float beta  = (float)(y-t0.y)/segment_height; // be careful with divisions by zero 
        Vec2i A = t0 + (t2-t0)*alpha; 
        Vec2i B = t0 + (t1-t0)*beta; 
        if (A.x>B.x) std::swap(A, B); 
        for (int j=A.x; j<=B.x; j++) { 
            image.set(j, y, color); // attention, due to int casts t0.y+i != A.y 
        } 
    } 
    for (int y=t1.y; y<=t2.y; y++) { 
        int segment_height =  t2.y-t1.y+1; 
        float alpha = (float)(y-t0.y)/total_height; 
        float beta  = (float)(y-t1.y)/segment_height; // be careful with divisions by zero 
        Vec2i A = t0 + (t2-t0)*alpha; 
        Vec2i B = t1 + (t2-t1)*beta; 
        if (A.x>B.x) std::swap(A, B); 
        for (int j=A.x; j<=B.x; j++) { 
            image.set(j, y, color); // attention, due to int casts t0.y+i != A.y 
        } 
    } 
```

![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/img/b1a0fce5f1.png)

This could be enough, but I dislike to see the same code twice. That is why we will make it a bit less readable, but more handy for modifications/maintaining:

```C++ 
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    if (t0.y==t1.y && t0.y==t2.y) return; // i dont care about degenerate triangles 
    // sort the vertices, t0, t1, t2 lower−to−upper (bubblesort yay!) 
    if (t0.y>t1.y) std::swap(t0, t1); 
    if (t0.y>t2.y) std::swap(t0, t2); 
    if (t1.y>t2.y) std::swap(t1, t2); 
    int total_height = t2.y-t0.y; 
    for (int i=0; i<total_height; i++) { 
        bool second_half = i>t1.y-t0.y || t1.y==t0.y; 
        int segment_height = second_half ? t2.y-t1.y : t1.y-t0.y; 
        float alpha = (float)i/total_height; 
        float beta  = (float)(i-(second_half ? t1.y-t0.y : 0))/segment_height; // be careful: with above conditions no division by zero here 
        Vec2i A =               t0 + (t2-t0)*alpha; 
        Vec2i B = second_half ? t1 + (t2-t1)*beta : t0 + (t1-t0)*beta; 
        if (A.x>B.x) std::swap(A, B); 
        for (int j=A.x; j<=B.x; j++) { 
            image.set(j, t0.y+i, color); // attention, due to int casts t0.y+i != A.y 
        } 
    } 
}
```

[Here’s the commit](https://github.com/ssloy/tinyrenderer/tree/024ad4619b824f9179c86dc144145e2b8b155f52) for drawing 2D triangles.



## The method i adopt for my code

While not being really complicated, the source code for the line sweeping is a bit messy. Moreover, it is really an old-school approach designed for mono-thread CPU programming. Let us take a look at the following pseudo-code:

```C++
triangle(vec2 points[3]) { 
    vec2 bbox[2] = find_bounding_box(points); 
    for (each pixel in the bounding box) { 
        if (inside(points, pixel)) { 
            put_pixel(pixel); 
        } 
    } 
}
```
Do you like it? I do. It is really easy to find a bounding box. It is certainly no problem to check whether a point belongs a 2D triangle (or any convex polygon). 

_Off Topic: if I have to implement a code to check whether a point belongs to a polygon, and this program will run on a plane, I will never get on this plane. Turns out, it is a surprisingly difficult task to solve this problem reliably. But here we just painting pixels. I am okay with that._

There is another thing i like about this pseudocode: a neophyte in programming accepts it with enthusiasm, more experienced programmers often chuckle: “What an idiot wrote it?”. And an expert in computer graphics programming will shrug his shoulders and say: “Well, that’s how it works in real life”. Massively parallel computations in thousands of threads (i’m talking about regular consumer computers here) change the way of thinking.

Okay, let us start: first of all we need to know what the [barycentric coordinates](https://en.wikipedia.org/wiki/Barycentric_coordinate_system) are. Given a 2D triangle ABC and a point P, all in old good Cartesian coordinates (xy). Our goal is to find barycentric coordinates of the point P with respect to the triangle ABC. It means that we look for three numbers (1 − u − v,u,v) such that we can find the point P as follows:

![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/index0x.png)

While being a bit frightening at the first glance, it is really simple: imagine that we put three weights (1 −u−v,u,v) at the vertices A, B and C, respectively. Then the barycenter of the system is exactly in the point P. We can say the same thing with other words: the point P has coordinates (u,v) in the (oblique) basis (A,![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/index1x.png),![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/index2x.png)):

![](http://www.loria.fr/~sokolovd/cg-course/02-triangles/index3x.png)