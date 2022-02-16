# Filling triangles

Hi, everyone. It’s me.

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/cfa0f3a9d9.png)

More precisely, it is a model of my face rendered in the program we will be creating in the next hour or two. Last time we drew the wire mesh of a three-dimensional model. This time, we will fill polygons, or rather triangles. In fact, OpenGL triangulates almost any polygon, so there’s no need to consider sophisticated cases.

_Let me remind you, this series of articles is designed to allow you to program yourself. When I said that in two hours you can draw a picture like the one above, I did not mean the time to read my code. It’s time for creating your code from scratch. My code is provided here purely to compare your (working) program with mine. I am an awful programmer, it is very likely that you are a better one. Do not simply copy-paste my code. Any comments and questions are welcome._

## Old-school method: Line sweeping

Thus, the task is to draw two-dimensional triangles. For motivated students it normally takes a couple of hours, even if they are bad programmers. Last time we saw Bresenham’s line drawing algorithm. Today’s task is to draw a filled triangle. Funny enough, but this task is not trivial. I don’t know why, but I know that it’s true. Most of my students struggle with this simple task.
So, the initial stub will look like this:
 
```C++
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    line(t0, t1, image, color); 
    line(t1, t2, image, color); 
    line(t2, t0, image, color); 
}

// ...

Vec2i t0[3] = {Vec2i(10, 70),   Vec2i(50, 160),  Vec2i(70, 80)}; 
Vec2i t1[3] = {Vec2i(180, 50),  Vec2i(150, 1),   Vec2i(70, 180)}; 
Vec2i t2[3] = {Vec2i(180, 150), Vec2i(120, 160), Vec2i(130, 180)}; 
triangle(t0[0], t0[1], t0[2], image, red); 
triangle(t1[0], t1[1], t1[2], image, white); 
triangle(t2[0], t2[1], t2[2], image, green);
```

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/41060d3251.png)

As usual, the appropriate commit is available on GitHub [here](https://github.com/ssloy/tinyrenderer/tree/7e46cc57fa3f5a41129d6b6fefe4e77f77b8aa84). The code is simple: I provide three triangles for the initial debugging of your code. If we invoke `line()` inside the triangle function, we’ll get the contour of the triangle. How to draw a filled triangle?

A good method of drawing a triangle must have the following features:

* It should be (surprise!) simple and fast.
* It should be symmetrical: the picture should not depend on the order of vertices passed to the drawing function.
* If two triangles have two common vertices, there should be no holes between them because of rasterization rounding.
* We could add more requirements, but let’s do with these ones. Traditionally a line sweeping is used:

1. Sort vertices of the triangle by their y-coordinates;
2. Rasterize simultaneously the left and the right sides of the triangle;
3. Draw a horizontal line segment between the left and the right boundary points.

At this point my students start to lose the firm ground: which segment is the left one, which one is right? Besides, there are three segments in a triangle... Usually, after this introduction I leave my students for about an hour: once again, reading my code is much less valuable than comparing your own code with mine.

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

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/3a5643f513.png)

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

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/d8e0575a00.png)

Note that the segments are not continuous. Last time when we drew straight lines we struggled to get continuous segments and here I did not bother with rotating the image (remember the xy swapping?). Why? We fill the triangles aftewards, that’s why. If we connect the corresponding pairs of points by horizontal lines, the gaps disappear:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/c1f95127ad.png)

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
}
```

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/b1a0fce5f1.png)

This could be enough, but I dislike to see the same code twice. That is why we will make it a bit less readable, but more handy for modifications/maintaining:

```C++ 
void triangle(Vec2i t0, Vec2i t1, Vec2i t2, TGAImage &image, TGAColor color) { 
    if (t0.y==t1.y && t0.y==t2.y) return; // I dont care about degenerate triangles 
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



## The method I adopt for my code

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

_Off Topic: if I have to implement some code to check whether a point belongs to a polygon, and this program will run on a plane, I will never get on this plane. Turns out, it is a surprisingly difficult task to solve this problem reliably. But here we just painting pixels. I am okay with that._

There is another thing I like about this pseudocode: a neophyte in programming accepts it with enthusiasm, more experienced programmers often chuckle: “What an idiot wrote it?”. And an expert in computer graphics programming will shrug his shoulders and say: “Well, that’s how it works in real life”. Massively parallel computations in thousands of threads (i’m talking about regular consumer computers here) change the way of thinking.

Okay, let us start: first of all we need to know what the [barycentric coordinates](https://en.wikipedia.org/wiki/Barycentric_coordinate_system) are. Given a 2D triangle ABC and a point P, all in old good Cartesian coordinates (xy). Our goal is to find barycentric coordinates of the point P with respect to the triangle ABC. It means that we look for three numbers (1 − u − v,u,v) such that we can find the point P as follows:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index0x.png)

While being a bit frightening at the first glance, it is really simple: imagine that we put three weights (1 −u−v,u,v) at the vertices A, B and C, respectively. Then the barycenter of the system is exactly in the point P. We can say the same thing with other words: the point P has coordinates (u,v) in the (oblique) basis (A,![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index1x.png),![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index2x.png)):

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index3x.png)

So, we have vectors ![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index4x.png), ![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index5x.png) and ![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index6x.png), we need to find two real numbers u and v respecting the following constraint:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index7x.png)


It is a simple vector equation, or a linear system of two equations with two variables:


![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index8x.png)


I am lazy and do not want to solve linear systems in a scholar way. Let us write it in matrix form:


![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/index9x.png)


It means that we are looking for a vector (u,v,1) that is orthogonal to (ABx,ACx,PAx) and (ABy,ACy,PAy) *at the same time*! I hope you see [where I am heading](https://en.wikipedia.org/wiki/Cross_product). That is a small hint: to find an intersection of two straight lines in a plane (that is exactly what we did here), it is sufficient to compute one cross product. By the way, test yourself: how do we find an equation of a line passing through two given points?

So, let us program our new rasterization routine: we iterate through all pixels of a bounding box for a given triangle. For each pixel we compute its barycentric coordinates. If it has at least one negative component, then the pixel is outside of the triangle. Probably it is more clear to see the program directly:

```C++
#include <vector> 
#include <iostream> 
#include "geometry.h"
#include "tgaimage.h" 
 
const int width  = 200; 
const int height = 200; 
 
Vec3f barycentric(Vec2i *pts, Vec2i P) { 
    Vec3f u = Vec3f(pts[2][0]-pts[0][0], pts[1][0]-pts[0][0], pts[0][0]-P[0])^Vec3f(pts[2][1]-pts[0][1], pts[1][1]-pts[0][1], pts[0][1]-P[1]);
    /* `pts` and `P` has integer value as coordinates
       so `abs(u[2])` < 1 means `u[2]` is 0, that means
       triangle is degenerate, in this case return something with negative coordinates */
    if (std::abs(u.z)<1) return Vec3f(-1,1,1);
    return Vec3f(1.f-(u.x+u.y)/u.z, u.y/u.z, u.x/u.z); 
} 
 
void triangle(Vec2i *pts, TGAImage &image, TGAColor color) { 
    Vec2i bboxmin(image.get_width()-1,  image.get_height()-1); 
    Vec2i bboxmax(0, 0); 
    Vec2i clamp(image.get_width()-1, image.get_height()-1); 
    for (int i=0; i<3; i++) { 
        bboxmin.x = std::max(0, std::min(bboxmin.x, pts[i].x));
	bboxmin.y = std::max(0, std::min(bboxmin.y, pts[i].y));

	bboxmax.x = std::min(clamp.x, std::max(bboxmax.x, pts[i].x));
	bboxmax.y = std::min(clamp.y, std::max(bboxmax.y, pts[i].y));
    } 
    Vec2i P; 
    for (P.x=bboxmin.x; P.x<=bboxmax.x; P.x++) { 
        for (P.y=bboxmin.y; P.y<=bboxmax.y; P.y++) { 
            Vec3f bc_screen  = barycentric(pts, P); 
            if (bc_screen.x<0 || bc_screen.y<0 || bc_screen.z<0) continue; 
            image.set(P.x, P.y, color); 
        } 
    } 
} 
 
int main(int argc, char** argv) { 
    TGAImage frame(200, 200, TGAImage::RGB); 
    Vec2i pts[3] = {Vec2i(10,10), Vec2i(100, 30), Vec2i(190, 160)}; 
    triangle(pts, frame, TGAColor(255, 0, 0)); 
    frame.flip_vertically(); // to place the origin in the bottom left corner of the image 
    frame.write_tga_file("framebuffer.tga");
    return 0; 
}
```

*barycentric()* function computes coordinates of a point P in a given triangle, we already saw the details. Now let us see how works *triangle()* function. First of all, it computes a bounding box, it is described by two points: bottom left and upper right. To find these corners we iterate through the vertices of the triangle and choose min/max coordinates. I also added a clipping of the bounding box with the screen rectangle to spare the CPU time for the triangles outside of the screen. Congratulations, you know how to draw a triangle!

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/0ba3f3e659f5feff80a78840fb927a71.png)

# Flat shading render

We already know how to draw a model with empty triangles. Let us fill them with a random color. This will help us to see how well we have encoded filling of triangles. Here is the code:
 
```C++
for (int i=0; i<model->nfaces(); i++) { 
    std::vector<int> face = model->face(i); 
    Vec2i screen_coords[3]; 
    for (int j=0; j<3; j++) { 
        Vec3f world_coords = model->vert(face[j]); 
        screen_coords[j] = Vec2i((world_coords.x+1.)*width/2., (world_coords.y+1.)*height/2.); 
    } 
    triangle(screen_coords[0], screen_coords[1], screen_coords[2], image, TGAColor(rand()%255, rand()%255, rand()%255, 255)); 
}
```

It is simple: just like before, we iterate through all the triangles, convert world coordinates to screen ones and draw triangles. I will provide the detailed description of various coordinate systems in my following articles. Current picture looks something like this:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/0c58d8a735.png)

Let us get rid of these clown-colors and put some lighting. Captain Obvious: ”At the same light intensity, the polygon is illuminated most brightly when it is orthogonal to the light direction.”

Let us compare:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/5371a416d1.jpg)

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/97e210ee08.jpg)

We get zero illumination if the polygon is parallel to the vector of light. To paraphrase: the intensity of illumination is equal to the scalar product of the light vector and the normal to the given triangle. The normal to the triangle can be calculated simply as the [cross product](https://en.wikipedia.org/wiki/Cross_product) of its two sides.

As a side note, at this course we will perform linear computations on the colors. However (128,128,128) color is not half as bright as (255, 255, 255). We are going to ignore gamma correction and tolerate the incorrectness of the brightness of our colors.

```C++
Vec3f light_dir(0,0,-1); // define light_dir

for (int i=0; i<model->nfaces(); i++) { 
    std::vector<int> face = model->face(i); 
    Vec2i screen_coords[3]; 
    Vec3f world_coords[3]; 
    for (int j=0; j<3; j++) { 
        Vec3f v = model->vert(face[j]); 
        screen_coords[j] = Vec2i((v.x+1.)*width/2., (v.y+1.)*height/2.); 
        world_coords[j]  = v; 
    } 
    Vec3f n = (world_coords[2]-world_coords[0])^(world_coords[1]-world_coords[0]); 
    n.normalize(); 
    float intensity = n*light_dir; 
    if (intensity>0) { 
        triangle(screen_coords[0], screen_coords[1], screen_coords[2], image, TGAColor(intensity*255, intensity*255, intensity*255, 255)); 
    } 
}
```

But the dot product can be negative. What does it mean? It means that the light comes from behind the polygon. If the scene is well modelled (it is usually the case), we can simply discard this triangle. This allows us to quickly remove some invisible triangles. It is called [Back-face culling](http://en.wikipedia.org/wiki/Back-face_culling).

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/02-triangle/d5223f9b93.png)

Note that the inner cavity of the mouth is drawn on top of the lips. That is because of our dirty clipping of invisible triangles: it works perfectly for convex shapes only. We will get rid of this artifact next time when we encode the z-buffer.

[Here’s](https://github.com/ssloy/tinyrenderer/tree/e1a3f2b0f9638fa6db9e0437c621132e1baa3fb1) the current version of the render. Do you find that the image of my face was more detailed? Well, I cheated a bit: there is a quarter million triangles in it vs. roughly a thousand in this artificial head model. But my face was indeed rendered with the above code. I promise you that in following articles we will add much more details to this image.
