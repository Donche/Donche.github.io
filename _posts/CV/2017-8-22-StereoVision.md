---
layout: post
title: 双目视觉大结
category: 计算机视觉
keywords: OpenCV,2017,C++
---

# 双目视觉大结

* 本来应该是很多个小结的，结果因为一懒再懒，一转眼代码都写完了还没有小结过，干脆直接大结好了，长点就长点，反正有目录，跟小结其实是差不多的。
* 源码都在我的[github](https://github.com/Donche/StereoVision)里面，包括有用的和没用的，可以随意挑需要的出来。立体视觉有几种实现方式都做了（Feature Matching并没有做完，虽说特征点匹配的还比较精准，但是倒推回去的R跟T很不靠谱，干脆就做到匹配完算了）。Block Matching的准确性还是挺好的，一般来说只要标定没问题，稍微调一下nDisparities就可以用了。
* 因为也是实验室做的项目的一部分，所以功能性的代码暂时就更新到这了，以后应该大概可能会改进一下。
* 其实这方面的博客已经够多了，我也参考了不少，所以很基础知识的就不复习了，就说下主要思路。
* 最近一直在看机器视觉机器学习还有机器人步态...乱七八糟看了一大堆，虽说拿笔写了点笔记但却没有怎么系统整理过，导致好久没有管博客了真是惭愧。所以这个写长点以示悔过。
* ~~我不生产代码，我只是代码的搬运工...~~

# 名词解释
* 内参
* 外参
* 特征点（由关键点和描述子组成）
* 视差
* 三维重建
* 标定
* 重投影

# 立体视觉的两种实现方法
目前我知道的主要是Block Matching 和Feature Matching。   
前者提前标定好，经过极线校正之后将左右摄像头画面调为同一高度，然后用一个个的窗口进行匹配，得到视差图之后利用外参进行三维重建，效果很不错，速度也快，缺点就是得事先标定好，中途摄像头相对位置稍微变一下就得重新标定（比如自己用两个摄像头随意摆桌上的那种就很麻烦）。   
后者是分别对左右两个图像取特征点，计算描述子然后匹配、筛选，得到一一对应的特征点后反推摄像头外参，同时将图像进行三角剖分，然后用推到的外参进行三维重建，速度比较慢，而且推出来的外参很不稳定，我用ORB推出来的外参就完全没法用，只好作罢。

所以，主要使用Block Matching进行三维重建，Feature Matching只做到匹配

# Block Matching
主要思路：1.标定 2.获取视差图 3.三维重建
## 标定
标定使用的标定板理论上可以使用任何图像，一般来说用的是标准的标定板，有chessboard、circles grid 和asymmetric circles grid这三种。需要注意的是，对于chessboard，对应的长和宽分别为两个方向上检测出来的角点数，也就是黑白方格数的值减一，square size 是两个角点之间的距离；对于circles grid， 长宽为两个方向上的圆的数量，square size 为最近两圆圆心距离；asymmetric circles grid 的square size 为水平方向两圆圆心距离的一半。     
OpenCV标定使用的是张正有的方法，所以一般来说十张不同姿态的标定板就可以标定出不错的结果了，我一般用20-30张，重投影误差可以维持在0.08-0.1之间。又一个需要注意的是，如果使用circles grid，不应该使标定板距离摄像头太近，否则误差会很大。

### 内参
首先来看一下标定主要需要的参数：
```c++
CV_EXPORTS_W double calibrateCamera( InputArrayOfArrays objectPoints,
                                     InputArrayOfArrays imagePoints, Size imageSize,
                                     InputOutputArray cameraMatrix, InputOutputArray distCoeffs,
                                     OutputArrayOfArrays rvecs, OutputArrayOfArrays tvecs,
                                     int flags = 0, TermCriteria criteria = TermCriteria(
                                        TermCriteria::COUNT + TermCriteria::EPS, 30, DBL_EPSILON) );
```

好吧，确实很多......一个一个来吧
* objectPoints：opencv手册里有解释。In the new interface it is a vector of vectors of calibration pattern points in the calibration pattern coordinate space (e.g. std::vector<std::vector<cv::Vec3f>>). 所以就是在标定板坐标上的需要检测点的坐标了。之前设定的高和宽还有square size 就是给这个用的。生成这个参数十分简单，opencv里有源码可以参考，大概长这样：
```c++
void StereoCalibration::calcBoardCornerPositions( vector<Point3f>& corners)
{
	corners.clear();

	for (int i = 0; i < boardSize.height; ++i)
		for (int j = 0; j < boardSize.width; ++j)
			corners.push_back(Point3f(float(j*squareSize), float(i*squareSize), 0));
}
```
* imagePoints:it is a vector of vectors of the projections of calibration pattern points。对，就是找出来的点在图像里的坐标。这个可以用```findChessboardCorners```或者```findCirclesGrid```找出来，就不举例了。
* imageSize：图像大小
* cameraMatrix： 输出摄像头内参
* distCoeffs： 输出相机畸变矩阵
* rvecs(tvecs)：Output vector of rotation(translation) vectors estimated for each pattern view，可以用来计算重投影误差。
* flag：~~立flag 用的，~~ 设置一些标定方法，比如```CV_CALIB_FIX_K4```。

### 外参
知道了内参之后，外参就比较好搞了，标定也是很准的。先看一下函数：
```c++
CV_EXPORTS_W double stereoCalibrate( InputArrayOfArrays objectPoints,
                                     InputArrayOfArrays imagePoints1, InputArrayOfArrays imagePoints2,
                                     InputOutputArray cameraMatrix1, InputOutputArray distCoeffs1,
                                     InputOutputArray cameraMatrix2, InputOutputArray distCoeffs2,
                                     Size imageSize, OutputArray R,OutputArray T, OutputArray E, OutputArray F,
                                     int flags = CALIB_FIX_INTRINSIC,
                                     TermCriteria criteria = TermCriteria(TermCriteria::COUNT+TermCriteria::EPS, 30, 1e-6) );
```
* objectPoints：跟单目的一样，用同一个变量就行
* imagePoints：两个图像的imagepoints，同上
* cameraMatrix、distCoeffs1：同上，直接把单目标定出来的结果放进去就可以
* imageSize：同上
* R：两个摄像头之间的旋转矩阵
* T：摄像头之间的平移矩阵
* E：本征矩阵
* F：基础矩阵
* flag：就是flag 啊

### 极线校正

为了匹配需要，计算出四个map用来对摄像头得到的图像进行remap，得到双目平行校正后的图像：
```c++
CV_EXPORTS_W void stereoRectify( InputArray cameraMatrix1, InputArray distCoeffs1,
                                 InputArray cameraMatrix2, InputArray distCoeffs2,
                                 Size imageSize, InputArray R, InputArray T,
                                 OutputArray R1, OutputArray R2,
                                 OutputArray P1, OutputArray P2,
                                 OutputArray Q, int flags = CALIB_ZERO_DISPARITY,
                                 double alpha = -1, Size newImageSize = Size(),
                                 CV_OUT Rect* validPixROI1 = 0, CV_OUT Rect* validPixROI2 = 0 );

CV_EXPORTS_W void initUndistortRectifyMap( InputArray cameraMatrix, InputArray distCoeffs,
                        InputArray R, InputArray newCameraMatrix,
                        Size size, int m1type, OutputArray map1, OutputArray map2 );
```

## 获取视差图
opencv2提供的BM算法主要有BM、SGBM和GC。   
然而我用的是OpenCV3，跟2有些不一样。不知道为什么，但是我们好像要失去GC了。   
没关系，对于我这个来说BM差不多够了。   
原理的话就不细说，其他博客有很多讲解的，我就直接说参数吧(以OpenCV3为准)。   
StereoBM 是继承的StereoMatcher，双方各有一些参数设置。因为是继承下来的，所以直接在StereoBM的实例里面设置就行。   
```c++
//StereoMatcher
CV_WRAP virtual void compute( InputArray left, InputArray right,
                              OutputArray disparity ) = 0;
CV_WRAP virtual void setMinDisparity(int minDisparity) = 0;
CV_WRAP virtual void setNumDisparities(int numDisparities) = 0;
CV_WRAP virtual void setBlockSize(int blockSize) = 0;
CV_WRAP virtual void setSpeckleWindowSize(int speckleWindowSize) = 0;
CV_WRAP virtual void setSpeckleRange(int speckleRange) = 0;
CV_WRAP virtual void setDisp12MaxDiff(int disp12MaxDiff) = 0;
//StereoBM
CV_WRAP virtual void setPreFilterType(int preFilterType) = 0;
CV_WRAP virtual void setPreFilterSize(int preFilterSize) = 0;
CV_WRAP virtual void setPreFilterCap(int preFilterCap) = 0;
CV_WRAP virtual void setTextureThreshold(int textureThreshold) = 0;
CV_WRAP virtual void setUniquenessRatio(int uniquenessRatio) = 0;
CV_WRAP virtual void setSmallerBlockSize(int blockSize) = 0;
CV_WRAP virtual void setROI1(Rect roi1) = 0;
CV_WRAP virtual void setROI2(Rect roi2) = 0;
```
一个一个来：   
* compute：算视差图咯，把八位单通道的图像（CV_8UC1）输进去，就得到视差图了。
* setMinDisparity：最小视差，默认值是 0
* setNumDisparities：最大视差值与最小视差值之差, 必须是 16 的整数倍。在相同外参的情况下，一般来说这个值越大可以检测到的最近距离越小（越近的物体有越大的视差值）
* setBlockSize：设置block 的大小，跟以前的SADWindowsize是一样的，值越大匹配到的块越大...一般应该在 5x5 至 21x21 之间，参数必须是奇数
* setSpeckleWindowSize：检查视差连通区域变化度的窗口大小, 值为 0 时取消 speckle 检查
* setSpeckleRange：视差变化阈值，当窗口内视差变化大于阈值时，该窗口内的视差清零
* setDisp12MaxDiff：左视差图（直接计算得出）和右视差图（通过cvValidateDisparity计算得出）之间的最大容许差异。超过该阈值的视差值将被清零。该参数默认为 -1，即不执行左右视差检查。


* setPreFilterType：预处理滤波器的类型，主要是用于降低亮度失真（photometric distortions）、消除噪声和增强纹理等
* setPreFilterSize：预处理滤波器窗口大小，容许范围是[5,255]，一般应该在 5x5..21x21 之间，参数必须为奇数值
* setPreFilterCap：预处理滤波器的截断值，预处理的输出值仅保留[-preFilterCap, preFilterCap]范围内的值
* setTextureThreshold：低纹理区域的判断阈值。如果当前窗口内所有邻居像素点的x导数绝对值之和小于指定阈值，则该窗口对应的像素点的视差值为 0
* setUniquenessRatio：视差唯一性百分比， 视差窗口范围内最低代价是次低代价的(1 + uniquenessRatio/100)倍时，最低代价对应的视差值才是该像素点的视差，否则该像素点的视差为 0
* setROI1，setROI2：左右视图的有效像素区域，一般由双目校正阶段的 cvStereoRectify 函数传递，也可以自行设定。一旦在状态参数中设定了 roi1 和 roi2，OpenCV 会通过cvGetValidDisparityROI 函数计算出视差图的有效区域，在有效区域外的视差值将被清零。


可见参数超级多，然而比较重要的只有SADWindowSize、numberOfDisparities 和 uniquenessRatio，其他的可以稍微少花点时间。我最终选择的参数是：
```c++
int preFilterSize = 13;
int preFilterCap = 24;
int minDisparity = -16;

int ndisparities = 8 * 16;
int SADWindowSize = 29;

int textureThreshold = 507;
int uniquenessRatio = 8;
int speckleWindowSize = 67;
int speckleRange = 14;
```


## 三维重建
就是用三角测量还原出来场景的三维点，OpenCV 用reprojectImageTo3D 实现：
```c++
CV_EXPORTS_W void reprojectImageTo3D( InputArray disparity,
                                      OutputArray 3dImage, InputArray Q,
                                      bool handleMissingValues = false,
                                      int ddepth = -1 );
```
得到的三维点可以导出到文件给MatLab画出来，或者用OpenGL实时显示（不知道为什么我这个OpenGL画出来的画风很诡异，大概是哪没搞明白，等好了再详细介绍这块吧）


# Feature Matching
主要思路 1.找特征点 2.匹配、筛选
## 特征点
常用的主要有SURF、ORB、SIFT。一般来说就计算速度上，ORB>SURF>SIFT；计算的复杂度则反过来。一般情况使用ORB是比较好的选择，经过筛选之后还会留下来不少匹配好的特征点。     
OpenCV3 里对特征点的实现方法改动比较大，需要用Feature2D 这个类来计算关键点和描述子，再用BFMatcher进行匹配。需要注意的是ORB使用Hamming distance作为描述子距离的度量，其他特征点可以使用不同的范数（如NORM_L2）。
具体特征点实现方法主要如下（省去了各种有效性的判断，只写了主要思路）：
```c++
Ptr<BFMatcher> matcher;
Ptr<Feature2D> feature_l, feature_r;

matcher = BFMatcher::create(NORM_L2);
feature_l = xfeatures2d::SURF::create();

feature_l->detectAndCompute(imgLeft, noArray(), key_points_l, descriptor_l);
feature_r->detectAndCompute(imgRight, noArray(), key_points_r, descriptor_r);
```

## 匹配和筛选
匹配主要有三种，KNNMatch、radiusMatch和普通的BFMatch。
* KNNMatch：Finds the k best matches for each descriptor from a query set.
* radiusMatch：For each query descriptor, finds the training descriptors not farther than the specified distance.
* match：Finds the best match for each descriptor from a query set.

都挺好理解的，我用的是KnnMatch，选择匹配最好的前两个，然后进行筛选。

筛选主要用到了两种手段，一个是ratioTest，一个是symmetryTest。
* ratioTest：
>"Therefore, for each feature point, we have two candidate matches in the other view. These are the two best ones based on the distance between their descriptors. If this measured distance is very low for the best match, and much larger for the second best match, we can safely accept the first match as a good one since it is unambiguously the best choice. Reciprocally, if the two best matches are relatively close in distance, then there exists a possibility that we make an error if we select one or the other. In this case, we should reject both matches."   

所以我取了0.7作为ratio，50作为最大距离进行筛选，得到每个特征点的最佳匹配，详情见源码。
* symmetryTest：进行双向的验证，如果左右两个图像中的一堆特征点互相匹配的话，就视为有效匹配。

以上两种方法结合起来就可以得到效果很不错的匹配结果，误匹配很少，速度也不慢。

# 总结
先到这里吧，其实还有很多可以写的，有时间了再补充。      
