---
title: 基于OpenCV对图片清晰度、色偏和亮度的检测（java版）
comments: true
date: 2019-04-26 14:57:09
tags: 
    - OpenCV
    - 图像清晰度、亮度、色偏检测
---
**由来：近期项目需要检测图片的亮度和色偏，但网上大多为用C实现的，没有java版本的，此篇为java版本对opencv的调用，谨以此献给CSDN的广大用户。**
### 一. 导入OpenCV所需依赖
**依赖下载：**[OpenCV运行环境下载(包含jar包和dll依赖库)](https://download.csdn.net/download/qq_34997906/10978639)
1. 在IDEA的项目模块下新建一个libs目录，将opencv-343.jar放进去，将opencv_java343.dll放到项目下。
<!--more-->
如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190227135253598.PNG?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0OTk3OTA2,size_16,color_FFFFFF,t_70)
注意：opencv_java343.dll文件很多时候会加载不了，放在系统的path路径下也是可以的，jdk目录以及windows32目录下都是可行的，如果有强迫症的话就放在项目下吧。



2. pom 文件依赖引入
```
<dependency>
     	<groupId>org.opencv</groupId>
        <artifactId>opencv</artifactId>
        <version>0.0.1</version>
        <scope>system</scope>
        <systemPath>${project.basedir}/libs/opencv-343.jar</systemPath>
</dependency>
```
### 二. 项目代码
**1. 色偏检测**
>**原理说明： 网上常用的一种方法是将RGB图像转变到CIE L*a*b*空间，其中L*表示图像亮度，a*表示图像红/绿分量，b*表示图像黄/蓝分量。通常存在色偏的图像，在a*和b*分量上的均值会偏离原点很远，方差也会偏小；通过计算图像在a*和b*分量上的均值和方差，就可评估图像是否存在色偏。计算CIE L*a*b*空间是一个比较繁琐的过程，好在OpenCV提供了现成的函数，因此整个过程也不复杂。**
```
/**
     * opencv 检测图片色偏
     * jpegFile:待检测的图片
     * calcCast 计算并返回一幅图像的色偏度以及，色偏方向
     * cast 计算出的偏差值，小于1表示比较正常，大于1表示存在色偏
     * da 红/绿色偏估计值，da大于0，表示偏红；da小于0表示偏绿
     * db 黄/蓝色偏估计值，db大于0，表示偏黄；db小于0表示偏蓝
     */
    import org.opencv.core.Core;
	import org.opencv.core.Mat;
	import org.opencv.imgcodecs.Imgcodecs;
	import org.opencv.imgproc.Imgproc;
	
    public static Boolean colorException(File jpegFile) {
        Mat srcImage = Imgcodecs.imread(jpegFile.getAbsolutePath());
        Mat dstImage = new Mat();
        //  将RGB图像转变到CIE L*a*b*
        Imgproc.cvtColor(srcImage, dstImage, Imgproc.COLOR_BGR2Lab);
        float a=0,b=0;
        int HistA[] = new int[256],HistB[] = new int[256];
        for(int i=0;i<256;i++)
        {
            HistA[i]=0;
            HistB[i]=0;
        }
        int size= (int)dstImage.total() * dstImage.channels();
        for(int i=0;i < dstImage.rows(); i++)
        {
            for(int j=0;j< dstImage.cols(); j++)
            {
                //在计算过程中，要考虑将CIEL*a*b*空间还原后同
                a+=(float)(dstImage.get(i,j)[1]-128);
                b+=(float)(dstImage.get(i,j)[2]-128);
//                int x=Math.abs(dstImage.ptr(i,j).get(1));
//                int y=Math.abs(dstImage.ptr(i,j).get(2));
                int x=(int)dstImage.get(i,j)[1];
                int y=(int)dstImage.get(i,j)[2];
                HistA[x]++;
                HistB[y]++;
            }
        }
        float  da=a/(float)(dstImage.rows() * dstImage.cols());
        float db=b/(float)(dstImage.rows() * dstImage.cols());
        float D= (float)Math.sqrt(da*da+db*db);
        float Ma=0,Mb=0;
        for(int i=0;i<256;i++)
        {
            //计算范围-128～127
            Ma+=Math.abs(i-128-da)*HistA[i];
            Mb+=Math.abs(i-128-db)*HistB[i];
        }
        Ma/=(float)(dstImage.rows() * dstImage.cols());
        Mb/=(float)(dstImage.rows() * dstImage.cols());
        float M=(float)Math.sqrt(Ma*Ma+Mb*Mb);
        float K=D/M;
        float cast =K;
        System.out.printf("色偏指数： %f\n",cast);
        if(cast>1.1) {
            System.out.printf("存在色偏\n");
            return true;
        }
        else {
            System.out.printf("不存在色偏\n");
            return false;
        }
    }
```
**2. 亮度检测**
>**原理说明：计算图片在灰度图上的均值和方差，当存在亮度异常时，均值会偏离均值点（可以假设为128），方差也会偏小；通过计算灰度图的均值和方差，就可评估图像是否存在过曝光或曝光不足。**
```
/**
     * opencv 检测图片亮度
     * brightnessException 计算并返回一幅图像的色偏度以及，色偏方向
     * cast 计算出的偏差值，小于1表示比较正常，大于1表示存在亮度异常；当cast异常时，da大于0表示过亮，da小于0表示过暗
     * 返回值通过cast、da两个引用返回，无显式返回值
     */
    import org.opencv.core.Core;
	import org.opencv.core.Mat;
	import org.opencv.imgcodecs.Imgcodecs;
	import org.opencv.imgproc.Imgproc;
	
    public static Integer brightnessException ( File jpegFile) {
        Mat srcImage = Imgcodecs.imread(jpegFile.getAbsolutePath());
        Mat dstImage = new Mat();
        // 将RGB图转为灰度图
        Imgproc.cvtColor(srcImage,dstImage, Imgproc.COLOR_BGR2GRAY);
        float a=0;
        int Hist[] = new int[256];
        for(int i=0;i<256;i++) {
            Hist[i] = 0;
        }
        for(int i=0;i<dstImage.rows();i++)
        {
            for(int j=0;j<dstImage.cols();j++)
            {
                //在计算过程中，考虑128为亮度均值点
                a+=(float)(dstImage.get(i,j)[0]-128);
                int x=(int)dstImage.get(i,j)[0];
                Hist[x]++;
            }
        }
        float da =  a/(float)(dstImage.rows()*dstImage.cols());
        System.out.println(da);
        float D =Math.abs(da);
        float Ma=0;
        for(int i=0;i<256;i++)
        {
            Ma+=Math.abs(i-128-da)*Hist[i];
        }
        Ma/=(float)((dstImage.rows()*dstImage.cols()));
        float M=Math.abs(Ma);
        float K=D/M;
        float cast = K;
        System.out.printf("亮度指数： %f\n",cast);
        if(cast>=1) {
            System.out.printf("亮度："+da);
            if(da > 0) {
                System.out.printf("过亮\n");
                return 2;
            } else {
                System.out.printf("过暗\n");
                return 1;
            }
        } else {
            System.out.printf("亮度：正常\n");
            return 0;
        }
    }
```
**3. 图片颜色检测**

```
	/**
     * opencv 检测图片颜色
     */
    public static void imageColor ( File jpegFile) {
        Mat srcImage = Imgcodecs.imread(jpegFile.getAbsolutePath());
        Mat dstImage = new Mat();
        Imgproc.cvtColor(srcImage,dstImage, Imgproc.COLOR_BGR2HSV);
        int i = 0 ,j = 0;
        loop:for( i=0;i<dstImage.rows();i++) {
            for(j=0;j<dstImage.cols();j++) {
                //在计算过程中，考虑128为亮度均值点
                double[] colorVec = dstImage.get(i,j);
                int x=(int)dstImage.get(i,j)[0];
                if((colorVec[0]>=0&&colorVec[0]<=180)
                        &&(colorVec[1]>=0&&colorVec[1]<=255)
                        &&(colorVec[2]>=0&&colorVec[2]<=46)) {
                    continue;
                }
                else  if((colorVec[0]>=0&&colorVec[0]<=180)
                        &&(colorVec[1]>=0&&colorVec[1]<=43)
                        &&(colorVec[2]>=46&&colorVec[2]<=220)){
                    continue;
                }
                else  if((colorVec[0]>=0&&colorVec[0]<=180)
                        &&(colorVec[1]>=0&&colorVec[1]<=30)
                        &&(colorVec[2]>=221&&colorVec[2]<=255)){
                    continue;
                }
                else {
                    System.out.println("彩色图像");
                    break loop;
                }
            }
        }
        if(i==dstImage.rows() && j==dstImage.cols()) {
            System.out.println("黑白图像");
        }
    }
```

**4. 清晰度检测**
网上用opencv检测的各个版本均为c++/c#写的，二者的库依赖和方法变量名都存在较大的差异，转换太过麻烦，此处提供一个javacv的写法，以解部分老哥的燃眉之急。
**4.1 javacv依赖引入（依赖jar包较大）**

```
<dependency>
        <groupId>org.bytedeco</groupId>
        <artifactId>javacv-platform</artifactId>
        <version>1.4.3</version>
</dependency>
```

```
import org.bytedeco.javacpp.opencv_core;
import org.bytedeco.javacpp.opencv_imgcodecs;
import org.bytedeco.javacpp.opencv_imgproc;

 	/**
     * javacv 检测图片清晰度
     * 标准差越大说明图像质量越好
     */
    public static void clarityException(File jpegFile){
        String path = "E:\\test\\";
        opencv_core.Mat srcImage = opencv_imgcodecs.imread(jpegFile.getAbsolutePath());
        opencv_core.Mat dstImage = new opencv_core.Mat();
        //转化为灰度图
        opencv_imgproc.cvtColor(srcImage, dstImage, opencv_imgproc.COLOR_BGR2GRAY);
        //在gray目录下生成灰度图片
        opencv_imgcodecs.imwrite(path+"gray-"+jpegFile.getName(), dstImage);

        opencv_core.Mat laplacianDstImage = new opencv_core.Mat();
        //阈值太低会导致正常图片被误断为模糊图片，阈值太高会导致模糊图片被误判为正常图片
        opencv_imgproc.Laplacian(dstImage, laplacianDstImage, opencv_core.CV_64F);
        //在laplacian目录下升成经过拉普拉斯掩模做卷积运算的图片
        opencv_imgcodecs.imwrite(path+"laplacian-"+jpegFile.getName(), laplacianDstImage);

        //矩阵标准差
        opencv_core.Mat stddev = new opencv_core.Mat();

        //求矩阵的均值与标准差
        opencv_core.meanStdDev(laplacianDstImage, new opencv_core.Mat(), stddev);
        // ((全部元素的平方)的和)的平方根
        // double norm = Core.norm(laplacianDstImage);
        // System.out.println("\n矩阵的均值：\n" + mean.dump());
        System.out.println(jpegFile.getName() + "矩阵的标准差：\n" + stddev.createIndexer().getDouble());
        // System.out.println(jpegFile.getName()+"平方根：\n" + norm);
    }
```
**最后的叮嘱：项目部署到服务器时，一定注意将opencv_java343.dll放在系统的path路径下，或者Tomcat的bin目录下。**