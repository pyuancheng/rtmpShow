### RTSP转RTMP

本次转流采用Centos+Nginx+FFmpeg实现，具体实现如下：

#### 1. 安装Ngxin

安装详细略（可以选择安装阿里的Tengine，官方[下载路径]([Download - The Tengine Web Server (taobao.org)](http://tengine.taobao.org/download.html)) ）

#### 2. 安装Nginx Rtmp模块

> nginx默认不支持rtmp流的转换，需要通过第三方扩展模块来实现转流。

##### 2.1 查看

``` shell
# 查看当前Nginx的编译安装使用了那些模块
/usr/local/nginx/sbin/nginx -V

[root@otwb-app03-uat //]# /usr/local/nginx/sbin/nginx -V
Tengine version: Tengine/2.3.1
nginx version: nginx/1.16.0
built by gcc 4.4.7 20120313 (Red Hat 4.4.7-17) (GCC)
built with OpenSSL 1.1.1a  20 Nov 2018
TLS SNI support enabled
configure arguments: --with-http_ssl_module
```

可以看出编译安装使用了configure arguments：**--with-http_ssl_module** 模块，还没有rtmp的模块。

##### 2.2 安装

```sh
# 获取
wget https://github.com/arut/nginx-rtmp-module/archive/master.zip

# 解压
unzip master.zip

# 进入Nginx的解压（源文件）目录，不是执行目录
cd /u01/nginx/tengine-2.3.1

# 增加下载好的模块，将上面查询的arguments复制下来，加上： --add-module=/home/work/software/nginx-rtmp-module-master，（注意模块的路径）
# 执行以下命令
./configure --with-http_ssl_module --add-module=/home/work/software/nginx-rtmp-module-master

# 接着执行make命令 注意：千万不要make install
make    

# 替换nginx的执行文件，上面编译之后执行文件已经改变了，需要将修改后的替换当前系统的nginx执行文件
# 进入nginx执行目录
cd /usr/local/nginx/sbin/nginx
# 备份
cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
# 替换，从安装tengine的解压目录中objs中复制到系统执行目录
cp /u01/nginx/tengine-2.3.1/objs/nginx /usr/local/nginx/sbin/nginx

# 重启nginx，查看，configure arguments已经多了rtmp的模块了。
/usr/local/nginx/sbin/nginx -V
[root@otwb-app03-uat //]# /usr/local/nginx/sbin/nginx -V
Tengine version: Tengine/2.3.1
nginx version: nginx/1.16.0
built by gcc 4.4.7 20120313 (Red Hat 4.4.7-17) (GCC)
built with OpenSSL 1.1.1a  20 Nov 2018
TLS SNI support enabled
configure arguments: --with-http_ssl_module --add-module=/u01/nginx/nginx-rtmp-module-master

```

##### 2.3 修改Nginx配置文件

修改nginx配置文件，添加如下内容并重新载入配置文件，rtmp标签和http同级

```sh
rtmp {  
   # 第一个转流地址
    server {  
        listen 1935;      #监听的端口号
        application hik01 {     #自定义的名字
            live on;  
       }  
    } 
     # 第二个转流地址
    server {  
        listen 1936;      #监听的端口号
        application hik02 {     #自定义的名字
            live on;  
       }  
    } 
    # 第N个转流地址
    server {  
        listen xxxx;      #监听的端口号
        application xxxx {     #自定义的名字
            live on;  
       }  
    } 
}
```

修改之后reload nginx文件。没问题即可下一步。



#### 3. 安装FFmpeg

##### 3.1 安装FFmpeg需要先安装其依赖：yasm 

```sh
yum install yasm -y
```

也可以从[官网]([Download - The Yasm Modular Assembler Project (tortall.net)](http://yasm.tortall.net/Download.html))源码安装

```sh
# 源码安装方式
# 编译
./configure
# 安装
make && make install
```



##### 3.2 安装FFmpeg

```sh
# 获取
wget https://ffmpeg.org/releases/ffmpeg-4.1.tar.bz2
# 解压
tar -xjvf ffmpeg-4.1.tar.bz2
# 查看
cd ffmpeg-4.1
# 编译
./configure
# 安装
make && make install

```



#### 4. 测试验证

##### 4.1 启用ffmpeg进行推流

> 以下命令需要修改rtsp流地址，rtmp地址以服务器实际配置为准，其他命令暂时复制即可。
>
> -rtsp_transport tcp 是将默认的udp协议转为tcp协议，可以一定程度上解决花屏（丢包）的问题。
>
> 详细的命令介绍，请看附录1

```sh
# 命令
ffmpeg -rtsp_transport tcp -i [rtsp流地址] flv -r 25 -s 1920*1080 -an [转换后的rtmp流地址]
# 实例
ffmpeg -rtsp_transport tcp -i rtsp://admin:123456@192.168.00.00 -f flv -r 25 -s 1920*1080 -an rtmp://localhost:1935/hik01/
# 后台运行，在命令前加nohup，后加 &
nohup ffmpeg -rtsp_transport tcp -i rtsp://admin:123456@192.168.00.00 -f flv -r 25 -s 1920*1080 -an rtmp://localhost:1935/hik01/ &
```

![image-20210114120033159](https://gitee.com/pongyc/picgo/raw/master/image-20210114120033159.png)



##### 4.2 利用VLC等视频工具验证rtmp流是否可用，VLC[下载]([Downloads - VideoLAN](http://get.videolan.org/vlc/3.0.11/win64/vlc-3.0.11-win64.exe))

![image-20210114120614452](https://gitee.com/pongyc/picgo/raw/master/image-20210114120614452.png)

![image-20210114120635212](https://gitee.com/pongyc/picgo/raw/master/image-20210114120635212.png)

![image-20210114120736334](https://gitee.com/pongyc/picgo/raw/master/image-20210114120736334.png)

#### 5. 网页端展示







#### 附录1 FFmpeg命令参数详解

##### 压缩视频

```sh
ffmpeg -i pingcap-intro-converted.mp4 -b:v 64k -r 20 -c:v libx264 -s 640x320 -strict -2 pingcap.mp4
```

##### 获取封面

```sh
ffmpeg -ss 00:00:10 -i test1.flv -f image2 -y test1.jpg
```

##### 屏幕类型

```sh
普屏4:3  320*240 640*480 
宽屏16:9  480*272 640*360 672*378 720*480 1024*600 1280*720 1920*1080 
```

**ffmpeg命令参数如下:**

| 参数名称             | 输入值                               | 备注                                                         |
| -------------------- | ------------------------------------ | ------------------------------------------------------------ |
| -i                   | ffmpmg -i pingcap-xxx.mp4            | 输入您要处理的视频文件路径                                   |
| -b：v $k -bufsize $k | -b：v 64k -bufsize 64k               | 要将输出文件的视频比特率设置为64 kbit / s                    |
| -r                   | ffmpeg -i input.avi -r 24 output.avi | 要强制输出文件的帧频为24 fps                                 |
| -c:v                 | -c:v libx264                         | ffmpeg -i input -c:v libx264 -preset slow -crf 22-c:a copy output.mkv |

##### 通用选项

```sh
-L license

-h 帮助

-fromats 显示可用的格式，编解码的，协议的。。。

-f fmt 强迫采用格式fmt

-I filename 输入文件

-y 覆盖输出文件

-t duration 设置纪录时间 hh:mm:ss[.xxx]格式的记录时间也支持

-ss position 搜索到指定的时间 [-]hh:mm:ss[.xxx]的格式也支持

-title string 设置标题

-author string 设置作者

-copyright string 设置版权

-comment string 设置评论

-target type 设置目标文件类型(vcd,svcd,dvd) 所有的格式选项（比特率，编解码以及缓冲区大小）自动设置 ，只需要输入如下的就可以了：
ffmpeg -i myfile.avi -target vcd /tmp/vcd.mpg

-hq 激活高质量设置

-itsoffset offset 设置以秒为基准的时间偏移，该选项影响所有后面的输入文件。该偏移被加到输入文件的时戳，定义一个正偏移意味着相应的流被延迟了 offset秒。 [-]hh:mm:ss[.xxx]的格式也支持
```

##### 视频选项



```sh
-b bitrate 设置比特率，缺省200kb/s

-r fps 设置帧频 缺省25

-s size 设置帧大小 格式为WXH 缺省160X128.下面的简写也可以直接使用：
Sqcif 128X96 qcif 176X144 cif 252X288 4cif 704X576

-aspect aspect 设置横纵比 4:3 16:9 或 1.3333 1.7777

-croptop size 设置顶部切除带大小 像素单位

-cropbottom size –cropleft size –cropright size

-padtop size 设置顶部补齐的大小 像素单位

-padbottom size –padleft size –padright size –padcolor color 设置补齐条颜色(hex,6个16进制的数，红:绿:兰排列，比如 000000代表黑色)

-vn 不做视频记录

-bt tolerance 设置视频码率容忍度kbit/s

-maxrate bitrate设置最大视频码率容忍度

-minrate bitreate 设置最小视频码率容忍度

-bufsize size 设置码率控制缓冲区大小

-vcodec codec 强制使用codec编解码方式。 如果用copy表示原始编解码数据必须被拷贝。

-sameq 使用同样视频质量作为源（VBR）

-pass n 选择处理遍数（1或者2）。两遍编码非常有用。第一遍生成统计信息，第二遍生成精确的请求的码率

-passlogfile file 选择两遍的纪录文件名为file
```

##### 高级选项



```sh
-g gop_size 设置图像组大小

-intra 仅适用帧内编码

-qscale q 使用固定的视频量化标度(VBR)

-qmin q 最小视频量化标度(VBR)

-qmax q 最大视频量化标度(VBR)

-qdiff q 量化标度间最大偏差 (VBR)

-qblur blur 视频量化标度柔化(VBR)

-qcomp compression 视频量化标度压缩(VBR)

-rc_init_cplx complexity 一遍编码的初始复杂度

-b_qfactor factor 在p和b帧间的qp因子

-i_qfactor factor 在p和i帧间的qp因子

-b_qoffset offset 在p和b帧间的qp偏差

-i_qoffset offset 在p和i帧间的qp偏差

-rc_eq equation 设置码率控制方程 默认tex^qComp

-rc_override override 特定间隔下的速率控制重载

-me method 设置运动估计的方法 可用方法有 zero phods log x1 epzs(缺省) full

-dct_algo algo 设置dct的算法 可用的有 0 FF_DCT_AUTO 缺省的DCT 1 FF_DCT_FASTINT 2 FF_DCT_INT 3 FF_DCT_MMX 4 FF_DCT_MLIB 5 FF_DCT_ALTIVEC

-idct_algo algo 设置idct算法。可用的有 0 FF_IDCT_AUTO 缺省的IDCT 1 FF_IDCT_INT 2 FF_IDCT_SIMPLE 3 FF_IDCT_SIMPLEMMX 4 FF_IDCT_LIBMPEG2MMX 5 FF_IDCT_PS2 6 FF_IDCT_MLIB 7 FF_IDCT_ARM 8 FF_IDCT_ALTIVEC 9 FF_IDCT_SH4 10 FF_IDCT_SIMPLEARM

-er n 设置错误残留为n 1 FF_ER_CAREFULL 缺省 2 FF_ER_COMPLIANT 3 FF_ER_AGGRESSIVE 4 FF_ER_VERY_AGGRESSIVE

-ec bit_mask 设置错误掩蔽为bit_mask,该值为如下值的位掩码 1 FF_EC_GUESS_MVS (default=enabled) 2 FF_EC_DEBLOCK (default=enabled)

-bf frames 使用frames B 帧，支持mpeg1,mpeg2,mpeg4

-mbd mode 宏块决策 0 FF_MB_DECISION_SIMPLE 使用mb_cmp 1 FF_MB_DECISION_BITS 2 FF_MB_DECISION_RD

-4mv 使用4个运动矢量 仅用于mpeg4

-part 使用数据划分 仅用于mpeg4

-bug param 绕过没有被自动监测到编码器的问题

-strict strictness 跟标准的严格性

-aic 使能高级帧内编码 h263+

-umv 使能无限运动矢量 h263+

-deinterlace 不采用交织方法

-interlace 强迫交织法编码 仅对mpeg2和mpeg4有效。当你的输入是交织的并且你想要保持交织以最小图像损失的时候采用该选项。可选的方法是不交织，但是损失更大

-psnr 计算压缩帧的psnr

-vstats 输出视频编码统计到vstats_hhmmss.log

-vhook module 插入视频处理模块 module 包括了模块名和参数，用空格分开
```

##### 音频选项

```sh
-ab bitrate 设置音频码率

-ar freq 设置音频采样率

-ac channels 设置通道 缺省为1

-an 不使能音频纪录

-acodec codec 使用codec编解码
```

##### 音视频捕获选项

```sh
-vd device 设置视频捕获设备。比如/dev/video0

-vc channel 设置视频捕获通道 DV1394专用

-tvstd standard 设置电视标准 NTSC PAL(SECAM)

-dv1394 设置DV1394捕获

-av device 设置音频设备 比如/dev/dsp
```

##### 高级选项

```sh
-map file:stream 设置输入流映射

-debug 打印特定调试信息

-benchmark 为基准测试加入时间

-hex 倾倒每一个输入包

-bitexact 仅使用位精确算法 用于编解码测试

-ps size 设置包大小，以bits为单位

-re 以本地帧频读数据，主要用于模拟捕获设备

-loop 循环输入流。只工作于图像流，用于ffserver测试
```



#### 参考博客

[Linux-Nginx+rtmp+ffmpeg搭建流媒体服务器 - 别来无恙- - 博客园 (cnblogs.com)](https://www.cnblogs.com/yanjieli/archive/2019/03/28/10615638.html)

[ffmpeg命令参数详解 - 简书 (jianshu.com)](https://www.jianshu.com/p/049d03705a81)

[CentOS6.8--安装Tengine_shx1133的专栏-CSDN博客](https://blog.csdn.net/shx1133/article/details/80876059)

[Linux 安装 ffmpeg | 温欣爸比的博客 (wxnacy.com)](https://wxnacy.com/2018/11/27/linux-install-ffmpeg/)