---
title: DeepStream5.0结合OpenCV4实现视频的分析和截图
date: 2020-05-15 16:58:14
author: 马捷径
img: https://img-blog.csdnimg.cn/20200326193416267.png
top: true
cover: true
coverImg: https://img-blog.csdnimg.cn/20200326193416267.png
toc: true
mathjax: true
summary: 在deepstream的基础上利用opencv4进行截图
keywords: 
  - DeepStream
  - Opencv4
  - NvBufSurface
  - 截图
categories: 视频分析
tags:
  - 视频分析
  - DeepStream
---
# 0. 前言
上次测试opencv结合deepstream4进行截图时，出现了一个[错误](https://blog.csdn.net/weixin_38369492/article/details/105418579#3__94)。当时在deepstream4中，虽然报错却仍能保证程序正常进行。但是deepstream5出来以后，迁移代码再跑时，这个错误就直接让程序崩掉了。找了半天没找到原因，所以改写了一下截图部分代码。

**思路**
[DeepStream结合OpenCV4实现视频的分析和截图（二）](https://blog.csdn.net/weixin_38369492/article/details/105418579)中的方式是取一帧数据，进行格式和参数转换，保存到另一个==NvBufSurface==结构中。然后再从该结构取数据保存到opencv的==mat==中，imwrite存图。问题出现在==NvBufSurface==转换上。

这次干脆放弃转换NvBufSurface，不再调用==NvBufSurfTransform==函数，而是先取数据到mat再转换格式。而是参考[DeepStream结合OpenCV4实现视频的分析和截图（一）](https://blog.csdn.net/weixin_38369492/article/details/105121729)中方法，稍作调整。

# 1. 环境
- Ubuntu 18.04
- CUDA 10.2
- CUDNN 7.6.5.32
- TensorRT 7.0.0.2
- DeepStream 5.0
- OpenCV 4.2

# 2. 方法概述
取出指定源帧数据->cudaMemcpy拷贝出帧数据->存到mat结构中->颜色空间转换。

相比（一）中代码，这里多了指定源，以及NV12转BGR。 

- **指定源**利用surface->surfaceList[batch_id]
- **格式转换**首先要查看surface原始的色彩空间，比如我在deepstream-app上测试的，infer后的色彩空间是NV12。

1. **NV12的Mat定义：**

注意是==height*3/2==，==CV_8UC1==
```cpp
cv::Mat frame = cv::Mat(frame_height * 3 / 2, frame_width, CV_8UC1, src_data, frame_step);
```
2. **OpenCV中NV12转BGR**

```cpp
cv::cvtColor(frame, out_mat, CV_YUV2BGR_NV12);
```

3. **图像压缩**

```cpp
float fx = 0.6;
float fy = 0.6; //图像压缩比例
cv::Size dsize = cv::Size(round(fx * out_mat.cols), round(fy * out_mat.rows));
cv::resize(source_mat, out_mat, dsize, 0, 0, cv::INTER_AREA); //重采样差值法进行图像压缩
```

# 3. Code

```cpp
batch_id = frame_meta->batch_id;
memset(&in_map_info, 0, sizeof(in_map_info));
if (!gst_buffer_map(buf, &in_map_info, GST_MAP_READ))
{
    g_print("Error: Failed to map gst buffer\n");
}
surface = (NvBufSurface *)in_map_info.data;
char *src_data = NULL;
src_data = (char *)malloc(surface->surfaceList[batch_id].dataSize);
if (src_data == NULL)
{
    g_print("Error: failed to malloc src_data \n");
}
#ifdef PLATFORM_TEGRA
NvBufSurfaceMap(surface, -1, -1, NVBUF_MAP_READ);
NvBufSurfacePlaneParams *pParams = &surface->surfaceList[batch_id].planeParams;
unsigned int offset = 0;
for (unsigned int num_planes = 0; num_planes < pParams->num_planes; num_planes++)
{
    if (num_planes > 0)
        offset += pParams->height[num_planes - 1] * (pParams->bytesPerPix[num_planes - 1] * pParams->width[num_planes - 1]);
    for (unsigned int h = 0; h < pParams->height[num_planes]; h++)
    {
        memcpy((void *)(src_data + offset + h * pParams->bytesPerPix[num_planes] * pParams->width[num_planes]),
               (void *)((char *)surface->surfaceList[batch_id].mappedAddr.addr[num_planes] + h * pParams->pitch[num_planes]),
               pParams->bytesPerPix[num_planes] * pParams->width[num_planes]);
    }
}
NvBufSurfaceSyncForDevice(surface, -1, -1);
NvBufSurfaceUnMap(surface, -1, -1);
#else
cudaMemcpy((void *)src_data,
           (void *)surface->surfaceList[batch_id].dataPtr,
           surface->surfaceList[batch_id].dataSize,
           cudaMemcpyDeviceToHost);
#endif
gint frame_width = (gint)surface->surfaceList[batch_id].width;
gint frame_height = (gint)surface->surfaceList[batch_id].height;
gint frame_step = surface->surfaceList[batch_id].pitch;
cv::Mat frame = cv::Mat(frame_height * 3 / 2, frame_width, CV_8UC1, src_data, frame_step);
// g_print("%d\n",frame.channels());
// g_print("%d\n",frame.rows);
// g_print("%d\n",frame.cols);
cv::Mat out_mat = cv::Mat(cv::Size(frame_width, frame_height), CV_8UC3);
cv::cvtColor(frame, out_mat, CV_YUV2BGR_NV12);
cv::imwrite(savefilename, out_mat);
if (src_data != NULL)
{
    free(src_data);
    src_data = NULL;
}
gst_buffer_unmap(buf, &in_map_info);
```