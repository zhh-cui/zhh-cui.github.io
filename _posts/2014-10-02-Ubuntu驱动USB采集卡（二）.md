---
layout: default
title: Ubuntu驱动USB采集卡（二）
---
- 捕获视频并保存到本地文件：test.raw

        sudo somagic-capture -i 1 | mplayer -vf yadif,screenshot -demuxer rawvideo -rawvideo "pal:format=uyvy:fps=25" -aspect 4:3 -

从这条命领可以得到两个信息：

- -rawvideo "pal:format=uyuv:fps=25" -aspect 4:3

    表明test.raw文件采集的原始格式是720x576@25fps的视频信号，保存格式是UYUV。
    
- 命令最后的"-"符号

    表示以上一条命令在终端的输出作为本命令的输入，这一点以后可以利用。


- 应该怎么读取test.raw

    UYUV的存储格式是：u12,y1,v12,y2;u34,y3,v34,y4;...
    
    每一帧大小是720x576像素，按照UYUV的格式存储（4字节表达2个像素），每一帧大小是829440字节。
    
    test.raw文件大小是170035200字节，总共包含205帧。
    
    YUV颜色空间到RGB颜色空间的转换公式是：
    
    r = y + 1.13983 * (v - 128.0);
    
    g = y - 0.39465 * (u - 128.0) - 0.58060 * (v - 128.0);
    
    b = y + 2.03211 * (u - 128.0);
    
- 源代码1

{% highlight c++ linenos %}
#include <iostream>
#include <opencv2/opencv.hpp>
#include <cstdio>

int main(int argc, char **argv) {
    FILE *fp = fopen(argv[1], "rb");
    if (!fp) {
        std::cout << "不能加载：" << argv[1] << std::endl;
        return 1;
    }

    unsigned char buf[829440];
    unsigned char val;
    char title[100];
    while (!feof(fp)) {
        sprintf(title, "%ld", ftell(fp) / 829440 + 1);

        for (int i = 0; i < 829440; ++i) {
            fscanf(fp, "%c", &val);
            buf[i] = val;
        }

        /// 720x576@25, 720x480@30
        cv::Mat bgr[3];
        bgr[0] = cv::Mat(576, 720, CV_8UC1, cv::Scalar(0));
        bgr[1] = cv::Mat(576, 720, CV_8UC1, cv::Scalar(0));
        bgr[2] = cv::Mat(576, 720, CV_8UC1, cv::Scalar(0));
        int row = 0;
        int col = 0;
        unsigned char y;
        unsigned char u;
        unsigned char v;
        for (int i = 0; i < 829440; i = i + 4) {
            u = buf[i];
            y = buf[i + 1];
            v = buf[i + 2];

            double r = (double)y + 1.13983 * ((double)v - 128.0);
            double g = (double)y - 0.39465 * ((double)u - 128.0) - 0.58060 * ((double)v - 128.0);
            double b = (double)y + 2.03211 * ((double)u - 128.0);
            bgr[0].at<unsigned char>(row, col) = cv::saturate_cast<unsigned char>(b);
            bgr[1].at<unsigned char>(row, col) = cv::saturate_cast<unsigned char>(g);
            bgr[2].at<unsigned char>(row, col) = cv::saturate_cast<unsigned char>(r);

            y = buf[i + 3];

            r = (double)y + 1.13983 * ((double)v - 128.0);
            g = (double)y - 0.39465 * ((double)u - 128.0) - 0.58060 * ((double)v - 128.0);
            b = (double)y + 2.03211 * ((double)u - 128.0);
            bgr[0].at<unsigned char>(row, col + 1) = cv::saturate_cast<unsigned char>(b);
            bgr[1].at<unsigned char>(row, col + 1) = cv::saturate_cast<unsigned char>(g);
            bgr[2].at<unsigned char>(row, col + 1) = cv::saturate_cast<unsigned char>(r);

            col = col + 2;
            if (col >= 720) {
                col = 0;
                row = row + 1;
            }
        }
        cv::Mat src;
        cv::merge(bgr, 3, src);

        cv::putText(src, title, cv::Point(0, 50), cv::FONT_HERSHEY_SIMPLEX, 1.0, cv::Scalar(255, 255 ,0));

        cv::imshow("img", src);
        cv::waitKey(40);
    }

    return 0;
}
{% endhighlight %}

调试命令：

        Debug/test ./test.raw
        
这里相当于离线调试。

- 源代码2

{% highlight c++ linenos %}
int main(int argc, char **argv) {
//  FILE *fp = fopen(argv[1], "rb");
//  if (!fp) {
//      std::cout << "不能加载：" << argv[1] << std::endl;
//      return 1;
//  }

    unsigned char buf[829440];
    unsigned char val;
    char title[10];
//  while (!feof(fp)) {
    int counter = 0;
    while (1) {
//      sprintf(title, "%ld", ftell(fp) / 829440 + 1);
        sprintf(title, "%d", counter++);

        for (int i = 0; i < 829440; ++i) {
//          fscanf(fp, "%c", &val);
            scanf("%c", &val);
            buf[i] = val;
        }
...
{% endhighlight %}

调试命令：

        sudo somagic-capture -i 1 | Debug/test -
        
这里相当于在线调试。

- 存在问题
    
    效率极端低下，每帧的解析过程消耗时间有长有短，回放画面时快时慢。
    
        
