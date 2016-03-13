The subject for today is [normal mapping](https://en.wikipedia.org/wiki/Normal_mapping). What is the main difference between normal mapping and Phong shading? The key is the density of information we have. For Phong shading we use normal vectors given per vertex of triangle mesh (and interpolate it inside triangles), whereas normal mapping textures provide dense information, dramatically improving rendering details.

Well, we have already applied normal mapping [in previous lesson](https://github.com/ssloy/tinyrenderer/wiki/Lesson-6:-Shaders-for-the-software-renderer), however we used the global system of coordinates to store the texture. Today we are talking about [tangent space](https://en.wikipedia.org/wiki/Frenet%E2%80%93Serret_formulas) normal mapping.

So, we have two textures, the left one is given in the global frame (direct transformation from RGB to XYZ normal), whereas the right one in the Frenet frame:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/nm_textures.jpg)

To use right texture, for each pixel we draw we compute tangent space (Frenet) frame. In this basis one vector (usually z) is orthogonal to our surface, and two others give a plane tangent to the current point. Then we read a (perturbed) normal vector from our texture, transform its coordinates from Frenet frame to the global system coordinates and we are done. Usually normal maps provide small perturbations of normal vectors, thus textures are in dominant blue color.

Well, why such a mess? Why not to use global system as we did before? Imagine we want to animate our model. For example, I took the black guy model and opened his mouth. It is obvious that normal vectors are to be modified.

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/global_vs_tangent.jpg)

Left image gives the head with open mouth, but unchanged (global frame) normal texture. Inspect closely the interior of the lower lip. The light coming directly to his face; when the mouth was closed the backside of the lower lip naturally was not lit. Now the mouth is open, but the lip is not lit... The right image was computed with tangent space normal mapping.

Therefore, if we have an animated model, then for correct normal mapping in global frame we need to have one texture per frame of the animation, whereas tangent space deforms accordingly to the model and we need one texture only!

Here is another example:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/global_vs_tangent_diablo.jpg)

These are textures for the Diablo model. Notice that only one hand is drawn in the texture, and only one side of the tail. The artist used the same texture for both arms and for both sides of the tail. It means that in the global coordinate system I can provide normal vectors either for the left side of the tail, either for the right one, but not for both! Same goes for the arms. And I need different information for the left and the right sides, for example, inspect left and right cheekbones in the left image, naturally normal vectors are pointing in the opposite directions!

Okay, let us finish with the motivation section and go straight ahead to the computations.

# Starting point, Phong shading

Итак, давайте посмотрим на [отправную точку](https://github.com/ssloy/tinyrenderer/tree/e30ff353121460557e29dced5708652171dbc7d2). Шейдер очень простой, это затенение Фонга.

```
struct Shader : public IShader {
    mat<2,3,float> varying_uv;  // triangle uv coordinates, written by the vertex shader, read by the fragment shader
    mat<3,3,float> varying_nrm; // normal per vertex to be interpolated by FS

    virtual Vec4f vertex(int iface, int nthvert) {
        varying_uv.set_col(nthvert, model->uv(iface, nthvert));
        varying_nrm.set_col(nthvert, proj<3>((Projection*ModelView).invert_transpose()*embed<4>(model->normal(iface, nthvert), 0.f)));
        Vec4f gl_Vertex = Projection*ModelView*embed<4>(model->vert(iface, nthvert));
        varying_tri.set_col(nthvert, gl_Vertex);
        return gl_Vertex;
    }

    virtual bool fragment(Vec3f bar, TGAColor &color) {
        Vec3f bn = (varying_nrm*bar).normalize();
        Vec2f uv = varying_uv*bar;

        float diff = std::max(0.f, bn*light_dir);
        color = model->diffuse(uv)*diff;
        return false;
    }
};
```

Вот результат работы шейдера:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/starting_point_a.jpg)

Для простоты обучения и отладки я уберу текстуру кожи и применю простейшую [регулярную сетку](https://github.com/ssloy/tinyrenderer/raw/master/obj/grid.tga) с горизонтальными красными и вертикальными синими линиями:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/starting_point_b.jpg)

Давайте посмотрим, как работает наш шейдер Фонга на примере этой картинки:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/grid_texture.png)

Итак, для каждой вершины треугольника у нас даны её координаты p, её текстурные координаты uv и нормальные вектора к вершинам n. Для отрисовки каждого пикселя растеризатор нам даёт барицентрические координаты пикселя (альфа, бета, гамма). Это означает, что текущий пиксель имеет пространственные координаты p = альфа p0 + бета p1  + гамма p2. Мы интерполируем текстурные координаты ровно так же, затем интерполируем и вектор нормали:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/f00.png)

Обратите внимание, что красные и синии линии - это изолинии u и v, соответственно. Итак, для каждой точки нашей поверхности у нас задаётся (скользящий) репер Френе, у которого ось x параллельна красным линиям, ось y параллельна синим, а z ортогональная поверхности. Именно в этом репере и задаются нормальные вектора.

# Вычисляем линейную функцию по трём её точкам

Итак, наша задача - для каждого рисуемого пикселя посчитать тройку векторов, задающую репер Френе. Давайте для начала отвлечёмся и представим, что в нашем пространстве задана линейная функция f, которая каждой точке (x,y,z) сопоставляет вещественное число f(x,y,z) = Ax + By + Cz + D. Единственное, что мы не знаем чисел (A,B,C,D), но знаем значение функции в вершинах некоторого треугольника (p0, p1, p2):

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/f01.png)

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/gradient_a.png)

Можно представлять себе, что f - это просто высота некоторой наклонной плоскости. Мы фиксируем три разные точки на плоскости и знаем значения высот в этих точках. Красные линии на треугольнике показывают изолинии высоты: изолиния для высоты f0, для высоты f0+1 метр, f0+2 метра и т.д. Для линейной функции все эти изолинии, очевидно, являются параллельными прямыми.

Что нам интересно это даже не столько направление изолиний, сколько направление, им ортогональное. Если мы передвигаемся вдоль какой-то изолинии, то высота не меняется (спасибо, капитан), если мы отклоняем направление чуть-чуть от изолинии, то высота начинает меняться, а самое большое изменение на единицу высоты будет тогда, когда мы будем передвигаться в направлении, ортогональном изолиниям.

Вспоминаем, что направление наискорейшего подъёма для какой-то функции есть не что иное, как её градиент. Для линейной функции  f(x,y,z) = Ax + By + Cz + D её градиент - это постоянный вектор (A, B, C). Логично, что он постоянный, так как любая точка плоскости наклонена одинаково. Напоминаю, что мы не знаем чисел (A,B,C). Мы знаем только значение нашей функции в трёх разных точках. Можем ли мы восстановить A, B и С? Конечно.

Итак, у нас есть три точки p0, p1, p2 и три значения функции f0, f1, f2. Нам интересно найти вектор (A,B,C), дающий направление наискорейшего роста функции f. Давайте для удобства будем рассматривать функцию g(p), которая задаётся как g(p) = f(p) - f(p0):

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/gradient_b.png)

Очевидно, что мы просто переместили нашу наклонную плоскость, не изменив её наклона, поэтому направление наискорейшего роста для функции g будет совпадат с направлением наискорейшего роста функции f.

Давайте перепишем определение функции g:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/f02.png)

Обратите внимание, что верхний индекс p^x - это координата x точки p, а не степень. То есть, функция g - это всего-навсего скалярное произведение вектора, соединяющего текущую точку p с точкой p0 и вектора (A,B,C). Но ведь мы по-прежнему не знаем (A,B,C)! Не страшно, сейчас их найдём.

Итак, что нам известно? Нам известно, что если от точки p0 мы пойдём в p2, то функция g будет равняться f2-f0. Иными словами, скалярное произведение между векторами p2-p0 и ABC равняется f2-f0. То же самое для dot (p1-p0,ABC)=f1-f0. То есть, мы ищем вектор (ABC), который одновременно ортогонален нормали к треугольнику и имеет эти два ограничения на скалярные произведения:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/f03.png)

Запишем то же самое в матричной форме:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/f04.png) 

То есть, мы получили матричное уравнение Ax = b, которое легко решается:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/f05.png) 

Обратите внимание, что я использовал литеру А в двух смыслах, значение должно быть ясно из контекста. То есть, наша 3x3 матрица A, помноженная на неизвестный вектор x=(A,B,C), даёт вектор b = (f1-f0, f2-f0, 0). Неизвестный вектор находится умножением матрицы, обратной к A, на вектор b.

Обратите внимание, что в матрице A нет ничего, что зависит от функции f! Там содержится только информация о геометрии треугольника.

# Вычисляем репер Френе и применяем карту (возмущения) нормалей

Итого, репер Френе - это тройка векторов (i,j,n), где n - это вектор нормали, а i и j могут быть подсчитаны следующим образом:

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/f06.png) 

[Вот финальный код](https://github.com/ssloy/tinyrenderer/tree/907bb561c38e7bd86db8d99678c0108f2e53d54d), использующий карты нормалей, заданные в касательном пространстве, а [тут](https://github.com/ssloy/tinyrenderer/commit/907bb561c38e7bd86db8d99678c0108f2e53d54d) можно найти изменения в коде по сравнению с тонировкой Фонга.

Там всё довольно прямолинейно, я вычисляю матрицу A:

```
        mat<3,3,float> A;
        A[0] = ndc_tri.col(1) - ndc_tri.col(0);
        A[1] = ndc_tri.col(2) - ndc_tri.col(0);
        A[2] = bn;
```

Затем вычисляю вектора репера Френе:

```
        mat<3,3,float> AI = A.invert();
        Vec3f i = AI * Vec3f(varying_uv[0][1] - varying_uv[0][0], varying_uv[0][2] - varying_uv[0][0], 0);
        Vec3f j = AI * Vec3f(varying_uv[1][1] - varying_uv[1][0], varying_uv[1][2] - varying_uv[1][0], 0);
```

Ну а однажды их вычислив, читаю нормаль из текстуры, и делаю простейшее преобразование координат из репера Френе в глобальный репер.
Если что, то преобразования координат [я уже описывал](https://github.com/ssloy/tinyrenderer/wiki/Lesson-5:-Moving-the-camera#change-of-basis-in-3d-space).

Вот финальный рендер, сравните детализацию с [тонировкой Фонга](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/starting_point_a.jpg):

![](https://raw.githubusercontent.com/ssloy/tinyrenderer/gh-pages/img/06b-tangent-space/normalmapping.jpg)

# Совет по отладке

Самое время вспомнить, [как отрисовываются линии](https://github.com/ssloy/tinyrenderer/wiki/Lesson-1:-Bresenham%E2%80%99s-Line-Drawing-Algorithm). Наложите на модель регулярную красно-синюю сетку и для всех вершин отрисуйте полученные вектора (i,j), они должны хорошо совпасть с направлением красно-синих линий текстуры.


Happy coding!

<spoiler title="Насколько вы были внимательны?">А заметили ли вы, что вообще у (плоского) треугольника вектор нормали постояннен, а я использовал интерполированную нормаль в последней строчке матрицы A? Почему я так сделал?</spoiler>