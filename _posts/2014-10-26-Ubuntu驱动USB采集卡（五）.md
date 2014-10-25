---
layout: default
title: Ubuntu驱动USB采集卡（五）
---
### 解决画面”破损“很严重 ###

- 首先怀疑是某个线程处理不够及时，导致丢失某些数据

测试capture线程，每个while循环平均耗时40毫秒，与somagic-capture命令的man手册能对应上，也跟《Ubuntu驱动USB采集卡（一）》中实时查看的mplayer命令输出能对应上，所以应该没有问题。

测试record线程，每个while循环平均耗时40毫秒，如果跳过等待信号量的语句（while中第一句），平均耗时20毫秒，应该也没有问题。

测试show线程，直接关闭该线程，只用mplayer查看videofile（即“test.avi”），问题仍然存在。

所以怀疑被否定。

- 其次怀疑是逐行扫描/隔行扫描的影响

逐帧查看test.avi，发现出现问题的帧，局部区域有明显的横向扫描线，所以立即怀疑是否逐行/隔行的问题。

但是，同样以“sudo somagic-capture -i 1”作为捕获命令，通过管道符给mplayer时，画面就没有问题，给我的程序时，画面就有问题。

所以怀疑被否定。

- 偶然找到关键

将record线程与show线程的同步改为信号量。

查看somagic-capture的man手册，发现参数“--iso-transfers”，可以增加并发传输的性能，降低某些人工影响，默认是5，尝试设定其他数字。

另外注意到一个细节，之前被忽略了，采集卡输出视频720x576@25fps，手册里要求调整为720x540@25fps一遍保持比例，mplayer是改成了768x576@25fps。我遵照手册的建议。

经过尝试，6以上就能在我这里解决问题，为保险起见，我设定为10，解决问题。

- 源代码5

{% highlight c++ linenos %}
#include <iostream>
#include <opencv2/opencv.hpp>
#include <cstdio>
#include <pthread.h>
#include <semaphore.h>

const int BufBlock = 829440;
const int BufSize = 5;
unsigned char *buf = NULL;
unsigned int wbuf = 0;
unsigned int rbuf = 0;
pthread_mutex_t mutex_buf[BufSize];
sem_t sem_buf, sem_show;

cv::Mat yuv(576, 720, CV_8UC3, cv::Scalar(0));
cv::Mat frame(576, 720, CV_8UC3, cv::Scalar(0));
cv::Mat image(540, 720, CV_8UC3, cv::Scalar(0));

int counter;
char title[10];

cv::VideoWriter videofile;

void* capture(void* parameter) {
    while (true) {
//      double start = cv::getTickCount();

        pthread_mutex_lock(mutex_buf + (wbuf % BufSize));

        unsigned char *data = buf + (wbuf % BufSize) * BufSize;
        for (int i = 0; i < BufBlock; ++i) {
            data[i] = getchar();
        }

        pthread_mutex_unlock(mutex_buf + (wbuf % BufSize));

        ++wbuf;

        sem_post(&sem_buf);

//      double stop = cv::getTickCount();
//      std::cout << (stop - start) * 1000.0 / cv::getTickFrequency() << " ms" << std::endl;
    }
}

void* record(void* parameter) {
    while (true) {
        sem_wait(&sem_buf);

//      double start = cv::getTickCount();

//      int cur = 0;
//      sem_getvalue(&sem_buf, &cur);
//      std::cout << "cur = " << cur << ", wbuf = " << wbuf << ", rbuf = " << rbuf << std::endl;

        pthread_mutex_lock(mutex_buf + (rbuf % BufSize));

        sprintf(title, "%d", ++counter);
        unsigned char *data = buf + (rbuf % BufSize) * BufSize;

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

        cv::cvtColor(yuv, frame, CV_YUV2BGR);
        cv::resize(frame, image, cv::Size(720, 540));
        cv::putText(frame, title, cv::Point(0, 50), cv::FONT_HERSHEY_SIMPLEX, 1.0, cv::Scalar(255, 255 ,0));

        videofile << image;

        sem_post(&sem_show);

        ++rbuf;

//      double stop = cv::getTickCount();
//      std::cout << (stop - start) * 1000.0 / cv::getTickFrequency() << " ms" << std::endl;
    }
}

void* show(void* parameter) {
    while (true) {
        sem_wait(&sem_show);
        cv::imshow("video", image);
        cv::waitKey(1);
    }
}

int main(int argc, char **argv) {
    buf = new unsigned char [BufSize * BufBlock];
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

