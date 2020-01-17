x264是一个开源的H.264编码库。本文介绍其在x86 linux的编译方法，并给出实例。  

# 一、编译  

x264编译需要使用到yasm，到http://www.tortall.net/projects/yasm/releases/下载，最新版本为1.3.1。编译命令如下：  

```console
# ./configure;make;make install
```

x264官方下载：http://www.videolan.org/developers/x264.html。下载压缩包名为：last_x264.tar.bz2。解压得到的文件名为x264-snapshot-xxxx。编译命令如下：  

```console
$ ./configure  --prefix=/home/latelee/bin/x264 --enable-static
$ make
$ make install
```

# 二、实例  

下面给出使用的示例。  

```c++
/**
x264库编码要使用以下库
-lpthread -ldl
要求头文件：stdint.h
ffprobe信息：
Input #0, h264, from 'test.h264':
  Duration: N/A, bitrate: N/A
    Stream #0:0: Video: h264 (High), yuv420p, 176x144, 25 fps, 25 tbr, 1200k tbn
, 50 tbc
*/
 
#include <stdio.h>
#include <stdlib.h>
 
#include <stdint.h> // for uint8_t in x264.h
 
#include "x264.h"
 
int x264_encode(const char* infile, int width, int height, int type, const char* outfile)
{
	FILE *fp_src = NULL;
	FILE *fp_dst = NULL;
    unsigned int luma_size = 0;
    unsigned int chroma_size = 0;
    int i_nal = 0;
    int i_frame = 0;
    unsigned int i_frame_size = 0;
 
    int csp = type;//X264_CSP_I420;
    x264_nal_t* nal = NULL;
    x264_t* handle = NULL;
    x264_picture_t* pic_in = NULL;
    x264_picture_t* pic_out = NULL;
    x264_param_t param;
 
    fp_src = fopen(infile, "rb");
    fp_dst = fopen(outfile, "wb");
    if(fp_src==NULL || fp_dst==NULL)
    {
        perror("Error open yuv files:");
        return -1;
    }
 
    x264_param_default(¶m);
 
    param.i_width  = width;
    param.i_height = height;
    /*
    //Param
    param.i_log_level  = X264_LOG_DEBUG;
    param.i_threads  = X264_SYNC_LOOKAHEAD_AUTO;
    param.i_frame_total = 0;
    param.i_keyint_max = 10;
    param.i_bframe  = 5;
    param.b_open_gop  = 0;
    param.i_bframe_pyramid = 0;
    param.rc.i_qp_constant=0;
    param.rc.i_qp_max=0;
    param.rc.i_qp_min=0;
    param.i_bframe_adaptive = X264_B_ADAPT_TRELLIS;
    param.i_fps_den  = 1;
    param.i_fps_num  = 25;
    param.i_timebase_den = param.i_fps_num;
    param.i_timebase_num = param.i_fps_den;
    */
    param.i_csp = csp;
    param.b_repeat_headers = 1;
    param.b_annexb = 1;
 
    // profile: high x264_profile_names定义见x264.h
    x264_param_apply_profile(¶m, x264_profile_names[2]);
 
    pic_in = (x264_picture_t*)malloc(sizeof(x264_picture_t));
    if (pic_in == NULL)
    {
        goto out;
    }
    pic_out = (x264_picture_t*)malloc(sizeof(x264_picture_t));
    if (pic_out == NULL)
    {
        goto out;
    }
    x264_picture_init(pic_out);
    x264_picture_alloc(pic_in, csp, param.i_width, param.i_height);
 
    handle = x264_encoder_open(¶m);
 
    // 计算有多少帧
    luma_size = param.i_width * param.i_height;
    fseek(fp_src, 0, SEEK_END);
    switch (csp)
    {
        case X264_CSP_I444:
            i_frame = ftell(fp_src) / (luma_size*3);
            chroma_size = luma_size;
            break;
        case X264_CSP_I420:
            i_frame = ftell(fp_src) / (luma_size*3/2);
            chroma_size = luma_size / 4;
            break;
        default:
            printf("Colorspace Not Support.\n");
            goto out;
    }
    fseek(fp_src, 0, SEEK_SET);
 
    printf("framecnt: %d, y: %d u: %d\n", i_frame, luma_size, chroma_size);
 
    // 编码
    for (int i = 0; i < i_frame; i++)
    {
        switch(csp)
        {
        case X264_CSP_I444:
        case X264_CSP_I420:
            i_frame_size = fread(pic_in->img.plane[0], 1, luma_size, fp_src);
            if (i_frame_size != luma_size)
            {
                printf("read luma failed %d.\n", i_frame_size);
                break;
            }
            i_frame_size = fread(pic_in->img.plane[1], 1, chroma_size, fp_src);
            if (i_frame_size != chroma_size)
            {
                printf("read chroma1 failed %d.\n", i_frame_size);
                break;
            }
            i_frame_size = fread(pic_in->img.plane[2], 1, chroma_size, fp_src);
            if (i_frame_size != chroma_size)
            {
                printf("read chrome2 failed %d.\n", i_frame_size);
                break;
            }
            break;
        default:
            printf("Colorspace Not Support.\n");
            return -1;
        }
        pic_in->i_pts = i;
 
        i_frame_size = x264_encoder_encode(handle, &nal, &i_nal, pic_in, pic_out);
        printf("encode frame: %5d framesize: %d nal: %d\n", i+1, i_frame_size, i_nal);
        if (i_frame_size < 0)
        {
            printf("Error encode frame: %d.\n", i+1);
            goto out;
        }
        #if 01
        else if (i_frame_size)
        {
            if(!fwrite(nal->p_payload, 1, i_frame_size, fp_dst))
                goto out;
        }
        #endif
        // 另一种做法
        #if 0
        // i_nal有可能为0，有可能多于1
        for (int j = 0; j < i_nal; j++)
        {
            fwrite(nal[j].p_payload, 1, nal[j].i_payload, fp_dst);
        }
        #endif
    }
 
    //flush encoder
    while(x264_encoder_delayed_frames(handle))
    //while(1)
    {
        static int cnt = 1;
        i_frame_size = x264_encoder_encode(handle, &nal, &i_nal, NULL, pic_out);
        printf("flush frame: %d framesize: %d nalsize: %d\n", cnt++, i_frame_size, i_nal);
        if(i_frame_size == 0)
        {
            break;
            //goto out;
        }
        #if 01
        else if(i_frame_size)
        {
            if(!fwrite(nal->p_payload, 1, i_frame_size, fp_dst))
                goto out;
        }
        #endif
        // 另一种做法
        #if 0
        // i_nal有可能为0，有可能多于1
        for (int j = 0; j < i_nal; j++)
        {
            fwrite(nal[j].p_payload, 1, nal[j].i_payload, fp_dst);
        }
        #endif
    }
 
out:
    x264_picture_clean(pic_in);
    if (handle)
    {
        x264_encoder_close(handle);
        handle = NULL;
    }
 
    if (pic_in)
        free(pic_in);
    if (pic_out)
        free(pic_out);
 
    fclose(fp_src);
    fclose(fp_dst);
 
    return 0;

}
```

使用方法：  

```console
x264_encode("../src/suzie_qcif_176x144_yuv420p.yuv", 176, 144, 1, "test.h264");
```

Makefile如下：  

```console
#
# (C) Copyleft 2011~2015
# Late Lee from http://www.latelee.org
# 
# A simple Makefile for *ONE* project(c or/and cpp file) in *ONE*  directory
#
# note: 
# you can put head file(s) in 'include' directory, so it looks 
# a little neat.
#
# usage: $ make
#        $ make debug=y
#
# log
#       2013-05-14 sth about debug...
###############################################################################
 
# !!!=== cross compile...
CROSS_COMPILE = 
 
CC = $(CROSS_COMPILE)gcc
CXX = $(CROSS_COMPILE)g++
AR = $(CROSS_COMPILE)ar
 
ARFLAGS = cr
RM = -rm -rf
MAKE = make
 
CFLAGS := -Wall
 
#****************************************************************************
# debug can be set to y to include debugging info, or n otherwise
debug          := n
 
#****************************************************************************
 
ifeq ($(debug), y)
CFLAGS += -ggdb -rdynamic
else
CFLAGS += -O2 -s
endif
 
# !!!===
DEFS = 
 
CFLAGS += $(DEFS)
 
LDFLAGS := $(LIBS)
 
# !!!===
INCDIRS := -I./include
 
# !!!===
CFLAGS += $(INCDIRS)
 
# !!!===
LDFLAGS += ./lib/libx264.a ./lib/libx265.a -lpthread -ldl
 
# !!!===
# source file(s), including c file(s) cpp file(s)
# you can also use $(wildcard *.c), etc.
SRC_C   := $(wildcard *.c)
SRC_CPP := $(wildcard *.cpp)
 
# object file(s)
OBJ_C   := $(patsubst %.c,%.o,$(SRC_C))
OBJ_CPP := $(patsubst %.cpp,%.o,$(SRC_CPP))
 
# !!!===
# executable file
target = a.out
 
###############################################################################
 
all: $(target)
 
$(target): $(OBJ_C) $(OBJ_CPP)
	@echo "Generating executable file..." $(notdir $(target))
	@$(CXX) $(CFLAGS) $^ -o $(target) $(LDFLAGS)
 
# make all .c or .cpp
%.o: %.c
	@echo "Compiling: " $(addsuffix .c, $(basename $(notdir $@)))
	@$(CC) $(CFLAGS) -c $< -o $@
 
%.o: %.cpp
	@echo "Compiling: " $(addsuffix .cpp, $(basename $(notdir $@)))
	@$(CXX) $(CFLAGS) -c $< -o $@
 
clean:
	@echo "Cleaning..."
	@$(RM) $(target)
	@$(RM) *.o *.back *~
 
.PHONY: all clean
```