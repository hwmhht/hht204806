# Introduction

Hello, let me introduce you my friend z-buffer of a black guy. He will help us get rid of visual artifacts of hidden faces removal we had in during the last lesson.

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/3f057a75601d8ac34555e72ea03ef711.png)

By the way, i'd like to mention that this model i use heavily in the course is created by [Vidar Rapp](https://se.linkedin.com/in/vidarrapp). He kindely granted me a permission to use it for teaching rendering basics and i vandalized it, but i promise you to give back the eyes to the black guy.

Well, back to the topic, in theory we could just draw all the triangles without discarding any. If we do it properly starting rear-to-front, the front facets will erase the back ones. It is called the [painter's algorithm](http://en.wikipedia.org/wiki/Painter%27s_algorithm). Unfortunately, it comes along with a high computational cost: for each camera movement we need to re-sort all the scene. And then there are dynamic scenes... And this is not even the main problem. The main problem is it is not always possible to determine the correct order.

# Let us try to render a simple scene

Imagine a simplest scene made of three triangles: the camera looks up-to-down, we project the colored triangles onto the white screen:

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/d493c52da4cabe9a057c26f696784956.png)

The render should look like this:

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/023668cb8ea97f59bf87d982c1e8b030.png)

Blue facet - is it behind or in front of the red one? The painter's algorithm does not work here. It is possible to split blue facet in two (one in front of the red facet and one behind). And then the one in front of the red one is to be split in two - one in front of the green triangle and one behind... I think you get the problem: in scenes with millions of triangles it is really expensive to compute. It is possible to use [BSP trees](https://en.wikipedia.org/wiki/Binary_space_partitioning) to get it done. By the way, this data structure is constant for moving camera, but it is really messy. And the life is too short to get it messy.

# Even simpler: let us loose a dimension. Y-buffer!

Let us loose a dimension for a while and to cut the above scene along the yellow plane:

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/d673f40bcadbe53f4b3cb29bbbcfb461.png)

I mean, now our scene is made of three line segments (intersection of the yellow plane and each of the triangles),
and the final render has a normal width but 1 pixel height:

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/3d4c4a1710b8e2558beb5c72ea52a61a.png)

As always, there is a [commit](https://github.com/ssloy/tinyrenderer/tree/d9c4b14c0d8c385937bc87cee1178f1e42966b7c) available. Our scene is two-dimensional, so it is easy to draw it using the line() function we programmed in the very first lesson.

```C++
    { // just dumping the 2d scene (yay we have enough dimensions!)
        TGAImage scene(width, height, TGAImage::RGB);

        // scene "2d mesh"
        line(Vec2i(20, 34),   Vec2i(744, 400), scene, red);
        line(Vec2i(120, 434), Vec2i(444, 400), scene, green);
        line(Vec2i(330, 463), Vec2i(594, 200), scene, blue);

        // screen line
        line(Vec2i(10, 10), Vec2i(790, 10), scene, white);

        scene.flip_vertically(); // i want to have the origin at the left bottom corner of the image
        scene.write_tga_file("scene.tga");
    }
```

This is how our 2D scene looks like if we look at it sideways:

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/20e9d8742d17979ec70e45cafacd63a5.png)

Let us render it. Recall that the render is 1 pixel height. In my source code i create images 16 pixels height for the ease of reading on high resolution screens. *rasterize()* function writes only in the first line of the image *render*

```C++
        TGAImage render(width, 16, TGAImage::RGB);
        int ybuffer[width];
        for (int i=0; i<width; i++) {
            ybuffer[i] = std::numeric_limits<int>::min();
        }
        rasterize(Vec2i(20, 34),   Vec2i(744, 400), render, red,   ybuffer);
        rasterize(Vec2i(120, 434), Vec2i(444, 400), render, green, ybuffer);
        rasterize(Vec2i(330, 463), Vec2i(594, 200), render, blue,  ybuffer);
```

So, i declared a magic array *ybuffer* with dimensions *(width, 1)*. This array is initialized with minus infinity. Then i call *rasterize()* function with this array and the image *render* as arguments. How does the function look like?

```C++
void rasterize(Vec2i p0, Vec2i p1, TGAImage &image, TGAColor color, int ybuffer[]) {
    if (p0.x>p1.x) {
        std::swap(p0, p1);
    }
    for (int x=p0.x; x<=p1.x; x++) {
        float t = (x-p0.x)/(float)(p1.x-p0.x);
        int y = p0.y*(1.-t) + p1.y*t;
        if (ybuffer[x]<y) {
            ybuffer[x] = y;
            image.set(x, 0, color);
        }
    }
}
```

It is really-really simple: i iterate through all x-coordinates between p0.x and p1.x and compute the corresponding y-coordinate of the segment. Then i check what we got in our array *ybuffer* with current x index. If the current y-value is closer to the camera than the value in the *ybuffer*, then i draw it on the screen and update the *ybuffer*.



![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/01694d604755b68c406998c03db374d9.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/65ddaf2b4d87f9b80127ecc6b02d0f72.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/6f081ac5fc77e2ec4bc733c945b16615.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/bae97132fc4ae67584b46b03d7350944.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/d6fdb1d49161923ac91796967afa766e.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/8f430d7de76bdcbda73b8de2986fbe49.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/24935d71a1b0023ee3cb48934fae175d.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/f93a1fc1cbaebb9c4670ae0003e62947.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/73714966ad4a4377b8c4df60bef03777.png)