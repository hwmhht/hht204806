# Perspective projection

In previous lessons we rendered our model in orthographic projection by simply forgetting the z-coordinate. The goal for today is to learn how to draw in perspective:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/39467dda61fdb644e68bdafc1e1f17f1.png)

# 2D geometry

## Linear transformations
A linear transformation on a plane can be represented by a corresponding matrix. If we take a point (x,y) then its transformation can be written as follows:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f00.svg)

The simplest (not degenerate) transformation is the identity, it does not move any point:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f01.svg)

Diagonal coefficients of the matrix give scaling along coordinate axes. Let us illustrate it, if we take the following transformation:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f02.svg)

Then the white object (the white square with one corner chopped) will be transformed into the yellow one. Red and green line segments give unit length vectors aligned with x and y, respectively:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/2aa8b671e124f1511c3b47a37c47f150.png)

All the images for this article were generated using [this code](https://github.com/ssloy/tinyrenderer/tree/a175be75a8a9a773bdfae7543a372e3bc859e02f).

Why do we bother with matrices? Because it is handy. First of all, in matrix form we can express a transformation of the entire object like this:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f03.svg)

In this expression the transformation matrix is the same as in the previous one, but the 2x5 matrix is nothing else but the vertices of our squarish object. We simply took all the vertices in an array, multiplied it by the transformation matrix and obtained the transformed object. Cool, is not it?

Well, the true reason hides here: very, very often we wish to transform our object with many transformations in a row. Imagine that in your source code you write transformation functions like

```C++
vec2 foo(vec2 p) return vec2(ax+by, cx+dy);
vec2 bar(vec2 p) return vec2(ex+fy, gx+hy);
[..]
for (each p in object) {
    p = foo(bar(p));
}
```

This code performs two linear transformations for each vertex of our object, and often we count those vertices in millions. And tens of transformations in a row is not a rare case, resulting in tens millions of operations, really expensive. In matrix form we can pre-multiply all the transformation matrices and to transform our object one time. For an expression with multiplications only we can put parentheses where we want, can we?

Okay, let us continue. We know that diagonal coefficients of the matrix scale our world along the coordinate axes. What other coefficients are responsible for? Let us consider the following transformation:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f04.svg)

Here is its action on our object:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/bb13159ffc0656ee622f9c4ebd108fed.png)

It is a simple shearing along the x-axis. Another anti-diagonal element shears our space along the y-axis. Thus, there are two base linear transformations on a plane: scaling and shearing. Many readers react: wait, what about rotations?!

It turns out that any rotation (around the origin) can be represented as a composite action of three shears, here the white object is transformed to the red one, then to the green one and finally to the blue:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/8723ca291b463b6eb44b9a91f5cbd26f.png)

But those are intricate details, to keep the things simple, a rotation matrix can be written directly (do you remember the pre-multiplication trick?):

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f05.svg)

We can multiply the matrices in any order, but let us remember that the multiplication for matrices is not commutative:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f06.svg)

It makes sense: to shear an object and then to rotate it is not the same as to rotate it and then to shear it!

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/7a85ee0ebed76be99ba9f97f0c89c5a4.png)

# 2D affine transformations

So, any linear transformation on a plane is a composition of scale and shear transformations. And it means that we can do any linear transformation we want, the origin wont ever move! Those possibilities are great, but if we can not perform simple translations, our life will be miserable. Can we? Okay, translations are not linear, no problem, let us try to append translations after performing the linear part:

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f07.svg)

This expression is really cool, we can rotate, we can scale, shear and translate. However. Let us recall that we are interested in composing multiple transformation, here is what a composition of two transformations look like (remember, we need to compose dozes of those?):

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f08.svg)

It is starting to look ugly even for a single composition, add more and things get even worse.

# Homogeneous coordinates

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f09.svg)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f10.svg)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/47cf05bf642df13f9b738e2c3040f648.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/0c054967a27e66bf020844118a1750d8.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/ed24b22a0542f9f930e0386c598d5a77.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/9e9658d91a6c8198606a8603012f048a.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f11.svg)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f12.svg)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/7f36ab01dad4a2937599de236c8d4d28.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/ff8f6a2130986fed747e55a26e054c6f.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/a7081e13ad5016aa33f87edb50b218f0.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/2b9f233797ca0a8b2d9d9f9750c29a36.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f13.svg)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f14.svg)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/525d3930435c3be900e4c7956edb5a1c.png)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f15.svg)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f16.svg)

![](http://webloria.loria.fr/~sokolovd/cg-course/04-perspective/img/f17.svg)
