
 　　这两个滤波器也是很久前就看过的，最近偶然翻起那本比较经典的matlab数字图像处理（冈萨雷斯）书，里面也提到了这个算法，觉得效果还行，于是也还是稍微整理下。


　　为了自己随时翻资料舒服和省事省时，这个算法的原理我们还是把他从别人的博客里搬过来吧： 


　　摘自：[图像处理基础(2\)：自适应中值滤波器(基于OpenCV实现)](https://github.com/wangguchangqing/p/6379646.html "发布于 2017-02-08 19:41") 


　　自适应的中值滤波器也需要一个矩形的窗口Sxy，和常规中值滤波器不同的是这个窗口的大小会在滤波处理的过程中进行改变（增大）。需要注意的是，滤波器的输出是一个像素值，该值用来替换点(x,y)处的像素值，点(x,y)是滤波窗口的中心位置。


     ![](https://img2024.cnblogs.com/blog/349293/202411/349293-20241104100101807-417025308.png)


原理说明


　　过程A的目的是确定当前窗口内得到中值Zmed是否是噪声。如果Zminmedmax，则中值Zmed不是噪声，这时转到过程B测试，当前窗口的中心位置的像素Zxy是否是一个噪声点。如果Zminxymax，则Zxy不是一个噪声，此时滤波器输出Zxy；如果不满足上述条件，则可判定Zxy是噪声，这是输出中值Zmed（在A中已经判断出Zmed不是噪声）。


　　如果在过程A中，得到则Zmed不符合条件Zminmedmax，则可判断得到的中值Zmed是一个噪声。在这种情况下，需要增大滤波器的窗口尺寸，在一个更大的范围内寻找一个非噪声点的中值，直到找到一个非噪声的中值，跳转到B；或者，窗口的尺寸达到了最大值，这时返回找到的中值，退出。


　　从上面分析可知，噪声出现的概率较低，自适应中值滤波器可以较快的得出结果，不需要去增加窗口的尺寸；反之，噪声的出现的概率较高，则需要增大滤波器的窗口尺寸，这也符合种中值滤波器的特点：噪声点比较多时，需要更大的滤波器窗口尺寸。


　　摘抄完成..............................................


　　这个过程理解起来也不是很困难，[图像处理基础(2\)：自适应中值滤波器(基于OpenCV实现)](https://github.com/wangguchangqing/p/6379646.html "发布于 2017-02-08 19:41"):[wgetCloud机场](https://tabijibiyori.org) 这个博客也给出了参考代码，不过我很遗憾的告诉大家，这个博客的效果虽然可以，但是编码和很多其他的博客一样，是存在问题的。


　　核心在这里：




```
for (int j = maxSize / 2; j < im1.rows - maxSize / 2; j++)
    {
        for (int i = maxSize / 2; i < im1.cols * im1.channels() - maxSize / 2; i++)
        {
            im1.at(j, i) = adaptiveProcess(im1, j, i, minSize, maxSize);
        }
    }
```


　　他这里的就求值始终是对同一份图，这样后续的处理时涉及到的领域实际上前面的部分已经是被修改的了，不符合真正的原理的。 至于为什么最后的结果还比较合适，那是因为这里的领域相关性不是特别强。


　　我这里借助于最大值和最小值滤波以及中值滤波，一个简单的实现如下所示：




```
/// 
/// 实现图像的自使用中值模糊。更新时间2015.3.11。
/// 参考：Adaptive Median Filtering Seminar Report By: PENG Lei (ID: 03090345)
/// 
/// 需要处理的源图像的数据结构。
/// 保存处理后的图像的数据结构。
/// 滤波的半径，有效范围[1,127]。
///  1: 能处理8位灰度和24位及32位图像。
///  2: 半径大于10以后基本没区别了。
int IM_AdaptiveMedianFilter(unsigned char* Src, unsigned char* Dest, int Width, int Height, int Stride, int MinRadius, int MaxRadius)
{
    int Channel = Stride / Width;
    if ((Src == NULL) || (Dest == NULL))                    return IM_STATUS_NULLREFRENCE;
    if ((Width <= 0) || (Height <= 0))                        return IM_STATUS_INVALIDPARAMETER;
    if ((MinRadius <= 0) || (MaxRadius <= 0))                return IM_STATUS_INVALIDPARAMETER;
    if ((Channel != 1) && (Channel != 3))                    return IM_STATUS_NOTSUPPORTED;
    int Status = IM_STATUS_OK;
    
    int Threshold = 10;
    if (MinRadius > MaxRadius)    IM_Swap(MinRadius, MaxRadius);

    bool AllProcessed = false;
    unsigned char* MinValue = (unsigned char*)malloc(Height * Stride * sizeof(unsigned char));
    unsigned char* MaxValue = (unsigned char*)malloc(Height * Stride * sizeof(unsigned char));
    unsigned char* MedianValue = (unsigned char*)malloc(Height * Stride * sizeof(unsigned char));
    unsigned char* Flag = (unsigned char*)malloc(Height * Width * sizeof(unsigned char));
    if ((MinValue == NULL) || (MaxValue == NULL) || (MedianValue == NULL) || (Flag == NULL))
    {
        Status = IM_STATUS_OUTOFMEMORY;
        goto FreeMemory;
    }
    memset(Flag, 0, Height * Width * sizeof(unsigned char));

    if (Channel == 1)
    {
        //    The median filter starts at size 3-by-3 and iterates up to size MaxRadius-by-MaxRadius
        for (int Z = MinRadius; Z <= MaxRadius; Z++)
        {
            Status = IM_MinFilter(Src, MinValue, Width, Height, Stride, Z);
            if (Status != IM_STATUS_OK)        goto FreeMemory;
            Status = IM_MaxFilter(Src, MaxValue, Width, Height, Stride, Z);
            if (Status != IM_STATUS_OK)        goto FreeMemory;        
            Status = IM_MedianBlur(Src, MedianValue, Width, Height, Stride, Z, 50);
            if (Status != IM_STATUS_OK)        goto FreeMemory;
        
            for (int Y = 0; Y < Height; Y++)
            {
                int Index = Y * Stride;
                int Pos = Y * Width;
                for (int X = 0; X < Width; X++, Index++, Pos++)
                {
                    if (Flag[Pos] == 0)
                    {
                        int Min = MinValue[Index], Max = MaxValue[Index], Median = MedianValue[Index];
                        if ((Median > Min + Threshold) && (Median < Max - Threshold))
                        {
                            int Value = Src[Index];
                            if ((Value > Min + Threshold) && (Value < Max- Threshold))
                            {
                                Dest[Index] = Value;
                            }
                            else
                            {
                                Dest[Index] = Median;
                            }
                            Flag[Pos] = 1;
                        }
                    }
                }
            }
            AllProcessed = true;
            for (int Y = 0; Y < Height; Y++)
            {
                int Pos = Y * Width;
                for (int X = 0; X < Width; X++)
                {
                    if (Flag[Pos + X] == 0)        //    检测是否每个点都已经处理好了
                    {
                        AllProcessed = false;
                        break;
                    }
                }
                if (AllProcessed == false) break;
            }
            if (AllProcessed == true) break;
        }

        /*    Output zmed for any remaining unprocessed pixels. Note that this
            zmed was computed using a window of size Smax-by-Smax, which is
            the final value of k in the loop.*/

        if (AllProcessed == false)
        {
            for (int Y = 0; Y < Height; Y++)
            {
                int Index = Y * Stride;
                int Pos = Y * Width;
                for (int X = 0; X < Width; X++)
                {
                    if (Flag[Pos + X] == 0)    Dest[Index + X] = Src[Index + X];
                }
            }
        }
    }
    else
    {

    }
FreeMemory:
    if (MinValue != NULL)        free(MinValue);
    if (MaxValue != NULL)        free(MaxValue);
    if (MedianValue != NULL)    free(MedianValue);
    if (Flag != NULL)            free(Flag);
    return Status;
}
```


　　注意这里，我们还做了适当的修改，增加了一个控制阈值Threshold，把原先的　if ((Median \> Min) \&\& (Median \< Max))


修改为：


　　if ((Median \> Min \+ Threshold) \&\& (Median \< Max \- Threshold))


　　也可以说是对应用场景的一种扩展，增加了函数的韧性。


　　当我们的噪音比较少时，这个函数会很快收敛，也就是不需要进行多次的计算的。


　　另外还有一个算法叫Conservative Smoothing，翻译成中文可以称之为保守滤波，这个算法在[https://homepages.inf.ed.ac.uk/rbf/HIPR2/csmooth.htm](https://github.com)有较为详细的说明，其基本原理是：



```
　This is accomplished by a procedure which first finds the minimum and maximum intensity values of all the pixels within a windowed region around the pixel in question. If the intensity of the central pixel lies within the intensity rangespread of its neighbors, it is passed on to the output image unchanged. However, if the central pixel intensity is greater than the maximum value, it is set equal to the maximum value; if the central pixel intensity is less than the minimum value,it is set equal to the minimum value. Figure 1 illustrates this idea.　　    ![](https://img2024.cnblogs.com/blog/349293/202411/349293-20241104104725937-1820696862.png)
```

 　　比如上图的3\*3领域，除去中心点之外的8个点其最大值为127，最小值是115，而中心点的值150大于这个最大值，所以中心点的值会被修改为127。


　　注意这里和前面的自适应中值滤波有一些不同。


　　1、虽然他也利用到了领域的最大值和最小值，但是这个领域是不包含中心像素本身的，这个和自适应中值是最大的区别。


       2、这个算法在满足某个条件时，不是用中值代替原始像素，而是用前面的改造的最大值或者最小值。


　　3、这个算法也可以改造成和自适应中值一样，不断的扩大半径。


　　4、对于同一个半径，这个函数多次迭代效果不会有区别。


　　一个简单的实现如下：




```
　　　　for (int X = 0; X < Width * Channel; X++, LinePD++)
        {
            int P0 = First[X], P1 = First[X + Channel], P2 = First[X + 2 * Channel];
            int P3 = Second[X], P4 = Second[X + Channel], P5 = Second[X + 2 * Channel];
            int P6 = Third[X], P7 = Third[X + Channel], P8 = Third[X + 2 * Channel];

            int Max0 = IM_Max(P0, P1);
            int Min0 = IM_Min(P0, P1);

            int Max1 = IM_Max(P2, P3);
            int Min1 = IM_Min(P2, P3);

            int Max2 = IM_Max(P5, P6);
            int Min2 = IM_Min(P5, P6);

            int Max3 = IM_Max(P7, P8);
            int Min3 = IM_Min(P7, P8);

            int Max = IM_Max(IM_Max(Max0, Max1), IM_Max(Max2, Max3));
            int Min = IM_Min(IM_Min(Min0, Min1), IM_Min(Min2, Min3));

            if (P4 > Max)
                P4 = Max;
            else if (P4 < Min)
                P4 = Min;
            LinePD[0] = P4;
        }
```


　　因为去除了中心点后进行的最大值和最小计算，这个算法如果要实现高效率的和半径无关的版本，还是需要做一番研究的，目前我尚没有做出成果。


　　我们拿标准的Lena图测试，使用matlab分别为他增加0\.02及0\.2范围的椒盐噪音，然后使用自适应中值滤波及保守滤波进行处理。


![](https://img2024.cnblogs.com/blog/349293/202411/349293-20241104121504686-1464516871.png)  ![](https://img2024.cnblogs.com/blog/349293/202411/349293-20241104121555948-1274966599.png)  ![](https://img2024.cnblogs.com/blog/349293/202411/349293-20241104121620032-306559977.png)


　　　　　　　　　　添加0\.02概率的椒盐噪音图　　　　　　　　　　　　　　　　　　　　　　　 　最小半径1，最大半径13时的去噪效果　　　　　　　　　　　　　　　　　　　　　　　　　　3\*3的保守滤波


![](https://img2024.cnblogs.com/blog/349293/202411/349293-20241104121750655-2005487322.png)  ![](https://img2024.cnblogs.com/blog/349293/202411/349293-20241104121801594-642799849.png)  ![](https://img2024.cnblogs.com/blog/349293/202411/349293-20241104121810852-1478397331.png)


　　　　　　添加0\.2概率的椒盐噪音图　　　　　　　　　　　　　　　　　　　　　　　　　　　 　　最小半径1，最大半径5时的去噪效果　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　3\*3的保守滤波


　　可以看到，自适应中值滤波在去除椒盐噪音的效果简直就是逆天，基本完美的复现了原图，有的时候我自己都不敢相信这个结果。而保守滤波由于目前我只实现了3\*3的版本，因此对于噪音比较集中的地方暂时无法去除，这个可以通过扩大半径和类似使用自适应中值的方式处理，而这种噪音的图像使用DCT去噪、小波去噪、NLM等等都不具有很好的效果，唯一能和他比拟的就只有蒙版和划痕里面小参数时的效果（这也是以中值为基础的）。所以去除椒盐噪音还是得靠中值相关的算法啊。


　　和常规的中值滤波器相比，自适应中值滤波器能够更好的保护图像中的边缘细节部分，当然代价就是增加了算法的时间复杂度。


