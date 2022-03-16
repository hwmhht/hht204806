```cpp
// Bresenham's 直线算法
// 具体实现参考 https://www.wikiwand.com/en/Bresenham%27s_line_algorithm#/All_cases
void draw_line(int x1, int y1, int x2, int y2, TGAImage &image, TGAColor color) {
    bool steep = false;
    if (std::abs(x1-x2) < std::abs(y1-y2)) {
        std::swap(x1, y1);
        std::swap(x2, y2);
        steep = true;
    }

    if (x1 > x2) {
        std::swap(x1, x2);
        std::swap(y1, y2);
    }

    int y = y1;
    int eps = 0;
    int dx = x2 - x1;
    int dy = y2 - y1;
    int yi = 1;

    if (dy < 0) {
        yi = -1;
        dy = -dy;
    }

    for (int x = x1; x <= x2; x++) {
        if (steep) {
            image.set(y, x, color);
        } else {
            image.set(x, y, color);
        }

        eps += dy;
        if((eps << 1) >= dx)  {
            y = y + yi;
            eps -= dx;
        }
    }
}
```