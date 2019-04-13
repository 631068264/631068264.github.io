---
layout:     post
rewards: false
title:    draw
categories:
- ml
tags:
- opencv
---
performance

# draw

```python
def draw():
    # 画图窗口大小设置为512*512 通常参数，可选1,3,4
    img = np.zeros((512, 512, 4), np.uint8)

    # 画直线，参数为绘图窗口，起点，终点，颜色，粗细，连通性（可选4联通或8联通）
    img = cv.line(img, (100, 0), (511, 511), (255, 0, 0), 5, 8)
    # 画矩形，参数为窗口，左上角坐标，右下角坐标，颜色，粗度
    img = cv.rectangle(img, (384, 0), (510, 128), (0, 255, 0), 3)
    # 画圆函数，参数为窗口，圆心，半径，颜色，粗度（如果粗度为-1标示实心）,连通性
    img = cv.circle(img, (447, 63), 50, (0, 0, 255), 1, 8)
    # 画椭圆函数，参数为窗口，椭圆中心，椭圆长轴短轴，椭圆逆时针旋转角度，椭圆起绘制起始角度，椭圆绘制结束角度，颜色，粗度，连通性
    img = cv.ellipse(img, (256, 256), (100, 50), 100, 36, 360, (0, 255, 255), 1, 8)

    # 画多边形
    pts = np.array([[10, 5], [20, 30], [70, 20], [50, 10]], np.int32)
    # 转换成三维数组，表示每一维里有一个点坐标
    pts = pts.reshape((-1, 1, 2))
    # 画多边形函数，参数为窗口，坐标，是否封闭，颜色
    img = cv.polylines(img, [pts], True, (0, 255, 255))

    # 字 参数为窗口，字符，字体位(左下角),字体类型(查询cv2.putText()函数的文档来查看字体),字体大小,常规参数，类似颜色，粗细，线条类型等等
    font = cv.FONT_HERSHEY_SIMPLEX
    cv.putText(img, 'OpenCV', (10, 500), font, 4, (255, 255, 255), 2, cv.LINE_AA)

    util.show(img)
```


# 鼠标 draw

监听鼠标事件 `cv.EVENT_*` `cv.setMouseCallback('image', draw_circle)`

```python
def mouse_draw():
    """双击画圆"""

    def draw_circle(event, x, y, flags, param):
        if event == cv.EVENT_LBUTTONDBLCLK:
            cv.circle(img, (x, y), 100, (255, 0, 0), -1)

    def draw_circle(event, x, y, flags, param):
        global ix, iy, drawing, mode
        # print(ix, iy, drawing, mode)
        if event == cv.EVENT_LBUTTONDOWN:
            drawing = True
            ix, iy = x, y
        elif event == cv.EVENT_MOUSEMOVE:
            if drawing:
                if mode:
                    cv.rectangle(img, (ix, iy), (x, y), (0, 255, 0), -1)
                else:
                    cv.circle(img, (x, y), 5, (0, 0, 255), -1)
        elif event == cv.EVENT_LBUTTONUP:
            drawing = False
            if mode:
                cv.rectangle(img, (ix, iy), (x, y), (0, 255, 0), -1)
            else:
                cv.circle(img, (x, y), 5, (0, 0, 255), -1)

    img = np.zeros((512, 512, 3), np.uint8)
    cv.namedWindow('image')
    cv.setMouseCallback('image', draw_circle)

    while True:
        cv.imshow('image', img)
        if cv.waitKey(20) & 0xFF == 27:
            break
    cv.destroyAllWindows()

```

# UI
```python
def bar_grb():
    def nothing(x):
        pass

    # Create a black image, a window
    img = np.zeros((300, 512, 3), np.uint8)
    cv.namedWindow('image')

    # create trackbars for color change
    cv.createTrackbar('R', 'image', 0, 255, nothing)
    cv.createTrackbar('G', 'image', 0, 255, nothing)
    cv.createTrackbar('B', 'image', 0, 255, nothing)
    # create switch for ON/OFF functionality
    switch = '0 : OFF \n1 : ON'
    cv.createTrackbar(switch, 'image', 0, 1, nothing)

    while True:
        cv.imshow('image', img)
        k = cv.waitKey(1) & 0xFF
        if k == 27:
            break
        # get current positions of four trackbars
        r = cv.getTrackbarPos('R', 'image')
        g = cv.getTrackbarPos('G', 'image')
        b = cv.getTrackbarPos('B', 'image')
        s = cv.getTrackbarPos(switch, 'image')
        if s == 0:
            img[:] = 0
        else:
            img[:] = [b, g, r]

    cv.destroyAllWindows()

```

