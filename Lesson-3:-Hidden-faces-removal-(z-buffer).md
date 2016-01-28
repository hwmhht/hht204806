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

Blue facet - is it behind or in front of the red one? The painter's algorithm does not work here. It is possible to split blue facet in two (one in front of the red facet and one behind). And then the one in front of the red one is to be split in two - one in front of the green triangle and one behind... I think you get the problem: in scenes with millions of triangles it is really expensive to compute. It is possible to use [BSP trees](https://en.wikipedia.org/wiki/Binary_space_partitioning), by the way, this data structure helps for moving camera, but it is really messy. And the life is too short to get it messy.

# Even simpler: let us loose a dimension. Y-buffer!

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/d673f40bcadbe53f4b3cb29bbbcfb461.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/3d4c4a1710b8e2558beb5c72ea52a61a.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/20e9d8742d17979ec70e45cafacd63a5.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/01694d604755b68c406998c03db374d9.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/65ddaf2b4d87f9b80127ecc6b02d0f72.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/6f081ac5fc77e2ec4bc733c945b16615.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/bae97132fc4ae67584b46b03d7350944.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/d6fdb1d49161923ac91796967afa766e.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/8f430d7de76bdcbda73b8de2986fbe49.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/24935d71a1b0023ee3cb48934fae175d.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/f93a1fc1cbaebb9c4670ae0003e62947.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/03-zbuffer/img/73714966ad4a4377b8c4df60bef03777.png)
