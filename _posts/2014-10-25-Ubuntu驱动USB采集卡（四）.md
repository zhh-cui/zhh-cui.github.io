---
layout: default
title: Ubuntu驱动USB采集卡（四）
---
- 第一个完整版本

用到了多线程、信号量、互斥量。给原始数据开辟了10帧缓冲区，循环使用。

主线程初始化变量，然后启动三个子线程，后面什么也不做，线程退出时的清理工作也没有，所以这只是功能测试版本。

- 源代码4

{% highlight c++ linenos %}
#include <iostream>
#include <opencv2/opencv.hpp>
#include <cstdio>
#include <pthread.h>
#include <semaphore.h>

const int BufBlock = 829440;
const int BufSize = 5;
unsigned char **buf = NULL;
unsigned int wbuf = 0;
unsigned int rbuf = 0;
pthread_mutex_t mutex_buf[BufSize], mutex_show;
sem_t sem_buf;

cv::Mat yuv(576, 720, CV_8UC3, cv::Scalar(0));
cv::Mat frame(576, 720, CV_8UC3, cv::Scalar(0));
int counter;
char title[10];

cv::VideoWriter videofile;

void* capture(void* parameter) {
    while (true) {
        pthread_mutex_lock(mutex_buf + (wbuf % BufSize));

        unsigned char *data = buf[wbuf % BufSize];
        for (int i = 0; i < BufBlock; ++i) {
            data[i] = getchar();
        }

        pthread_mutex_unlock(mutex_buf + (wbuf % BufSize));

        sem_post(&sem_buf);

        ++wbuf;
    }
}

void* record(void* parameter) {
    while (true) {
        sem_wait(&sem_buf);

        int cur = 0;
        sem_getvalue(&sem_buf, &cur);
        std::cout << "Current buffer: " << cur << ", wbuf = " << wbuf << ", rbuf = " << rbuf << std::endl;

        pthread_mutex_lock(mutex_buf + (rbuf % BufSize));

        sprintf(title, "%d", ++counter);
        unsigned char *data = buf[rbuf % BufSize];

        int row = 0;
        int col = 0;
        for (int i = 0; i < BufBlock; i = i + 4) {
            yuv.at<cv::Vec3b>(row, col) = cv::Vec3b(data[i + 1], data[i], data[i + 2]);  /// Y1, U1, V1
            yuv.at<cv::Vec3b>(row, col + 1) = cv::Vec3b(data[i + 3], data[i], data[i + 2]);  /// Y2, U2=U1, V2=V1

            col = col + 2;
            if (col >= 720) {
                col = 0;
                row = row + 1;
            }
        }

        pthread_mutex_unlock(mutex_buf + (rbuf % BufSize));

        pthread_mutex_lock(&mutex_show);
        cv::cvtColor(yuv, frame, CV_YUV2BGR);
        cv::putText(frame, title, cv::Point(0, 50), cv::FONT_HERSHEY_SIMPLEX, 1.0, cv::Scalar(255, 255 ,0));
        pthread_mutex_unlock(&mutex_show);
        videofile << frame;

        ++rbuf;
    }
}

void* show(void* parameter) {
    while (true) {
        pthread_mutex_lock(&mutex_show);
        cv::imshow("frame", frame);
        pthread_mutex_unlock(&mutex_show);
        cv::waitKey(40);
    }
}

int main(int argc, char **argv) {
    buf = new unsigned char * [BufSize];
    for (int i = 0; i < BufSize; ++i) {
        buf[i] = new unsigned char [BufBlock];
    }
    sem_init(&sem_buf, 0, 0);
    pthread_t captureid, recordid, showid;
    counter = 0;

    /// 'D', 'I', 'V', 'X'
    /// 'F', 'M', 'P', '4'
    videofile = cv::VideoWriter("test.avi", CV_FOURCC('F', 'M', 'P', '4'), 25.0, cv::Size(720, 576));

    int status = pthread_create(&captureid, NULL, capture, NULL);
    if (status != 0) {
        std::cerr << "不能创建capture线程: " << strerror(status);
        return 0;
    }

    status = pthread_create(&recordid, NULL, record, NULL);
    if (status != 0) {
        std::cerr << "不能创建record线程: " << strerror(status);
        return 0;
    }

    status = pthread_create(&showid, NULL, show, NULL);
    if (status != 0) {
        std::cerr << "不能创建show线程: " << strerror(status);
        return 0;
    }

    pthread_join(recordid, (void **)0);
    pthread_join(captureid, (void **)0);
    pthread_join(showid, (void **)0);

    return 0;
}
{% endhighlight %}

功能基本实现，但是发现画面”破损“很严重。

