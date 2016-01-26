# Triangle rasterization and back-face culling

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