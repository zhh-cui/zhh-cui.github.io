---
layout: default
title: Ubuntu驱动USB采集卡（三）
---
- 速度测试

“源代码1”的第22行改为：

{% highlight c++ linenos %}
double start = cv::getTickCount();
{% endhighlight %}

第64行改为：

{% highlight c++ linenos %}
double stop = cv::getTickCount();
std::cout << (stop - start) * 1000.0 / cv::getTickFrequency() << std::endl;
{% endhighlight %}

测试test.raw文件，每帧平均耗时44ms。

- 源代码3

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
    char title[10];
    while (!feof(fp)) {
        sprintf(title, "%ld", ftell(fp) / 829440 + 1);

        for (int i = 0; i < 829440; ++i) {
            fscanf(fp, "%c", &val);
            buf[i] = val;
        }

        double start = cv::getTickCount();

        /// 720x576@25, 720x480@30
        cv::Mat yuv(576, 720, CV_8UC3, cv::Scalar(0));
        int row = 0;
        int col = 0;
        for (int i = 0; i < 829440; i = i + 4) {
            yuv.at<cv::Vec3b>(row, col) = cv::Vec3b(buf[i + 1], buf[i], buf[i + 2]);  /// Y1, U1, V1
            yuv.at<cv::Vec3b>(row, col + 1) = cv::Vec3b(buf[i + 3], buf[i], buf[i + 2]);  /// Y2, U2=U1, V2=V1

            col = col + 2;
            if (col >= 720) {
                col = 0;
                row = row + 1;
            }
        }

        cv::Mat src;
        cv::cvtColor(yuv, src, CV_YUV2BGR);

        cv::putText(src, title, cv::Point(0, 50), cv::FONT_HERSHEY_SIMPLEX, 1.0, cv::Scalar(255, 255 ,0));

        double stop = cv::getTickCount();
        std::cout << (stop - start) * 1000.0 / cv::getTickFrequency() << std::endl;

        cv::imshow("img", src);
        cv::waitKey(1);
    }

    return 0;
}
{% endhighlight %}

测试test.raw文件，每帧平均耗时17ms。

