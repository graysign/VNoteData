# FFMPEG-桌面图像的采集

```c
#include <iostream>

extern "C"{
#include <libavformat/avformat.h>
#include <libavdevice/avdevice.h>
#include <libswscale/swscale.h>
#include <libswresample/swresample.h>
#include <libavutil/audio_fifo.h>
#include <libavutil/time.h>
#include <stdio.h>
}
using namespace std;

static AVFrame *alloc_picture(enum AVPixelFormat pix_fmt, int width, int height) {
    AVFrame *picture;
    int ret;

    picture = av_frame_alloc();
    if (!picture)
        return NULL;
    picture->format = pix_fmt;
    picture->width = width;
    picture->height = height;

    ret = av_frame_get_buffer(picture, 0);
    if (ret < 0) {
        fprintf(stderr, "Could not allocate frame data.\n");
        return NULL;
    }
    return picture;
}

// 图像的采集
static int test1(){
    // 创建一个文件管理器
    AVFormatContext *formatCtx = avformat_alloc_context();
    AVInputFormat *ifmt = av_find_input_format("avfoundation");

    // 配置流的参数
    AVDictionary *options = NULL;
//    av_dict_set(&options, "video_size". "1920*1080",0);
    av_dict_set(&options, "framerate", "15", 0);

    // 打开桌面流
    if (avformat_open_input(&formatCtx, "1", ifmt, &options) != 0){
        printf("open input device fail\n");
        return -1;
    }
    // 查找设个设备里面的媒体信息
    if (avformat_find_stream_info(formatCtx, NULL)<0)
    {
        printf("找不到媒体流信息!\n");
        return -1;
    }
    if(formatCtx->streams[0]->codecpar->codec_type != AVMEDIA_TYPE_VIDEO)
    {
        printf("流信息格式错误！");
    }
    // 找到解码器
    AVCodec *codec = avcodec_find_decoder(formatCtx->streams[0]->codecpar->codec_id);
    if (codec ==NULL){
        printf("codec not found.\n");
        return -1;
    }

    AVCodecContext *ctx = avcodec_alloc_context3(codec);
    // 进行解码
    avcodec_parameters_to_context(ctx, formatCtx->streams[0]->codecpar);
    if (avcodec_open2(ctx, codec, NULL)<0){
        printf("不能打开解码器");
        return -1;
    }

    // 初始化图像
    AVFrame *frame = alloc_picture(
            (AVPixelFormat)formatCtx->streams[0]->codecpar->format,
            formatCtx->streams[0]->codecpar->width,
            formatCtx->streams[0]->codecpar->height
            );
    // 图像的缩放
    SwsContext *img_convert_ctx = sws_getContext(
            formatCtx->streams[0]->codecpar->width,
            formatCtx->streams[0]->codecpar->height,
            (AVPixelFormat)formatCtx->streams[0]->codecpar->format,
            formatCtx->streams[0]->codecpar->width,
            formatCtx->streams[0]->codecpar->height,
            AV_PIX_FMT_BGR24,
            SWS_BICUBIC,
            NULL,
            NULL,
            NULL
    );
    AVFrame *bgrFrame = alloc_picture(
            AV_PIX_FMT_BGR24,
            formatCtx->streams[0]->codecpar->width,
            formatCtx->streams[0]->codecpar->height
            );

    while (true){


    AVPacket packet = {0};
    av_init_packet(&packet);
    if(av_read_frame(formatCtx, &packet) >= 0){

        // 发送包到解码器
        avcodec_send_packet(ctx, &packet);
        if (avcodec_receive_frame(ctx, frame)<0)
        {
            printf("Decode Error\n");
        } else{
            printf("采集到图片\n");
            sws_scale(img_convert_ctx,
                    (const unsigned char * const *)frame->data,
                    frame->linesize,
                    0,
                    frame->height,
                    bgrFrame->data,
                    bgrFrame->linesize
                    );

        }
    }
    av_packet_unref(&packet);
    }

    // 资源释放
    av_frame_free(&frame);
    av_frame_free(&bgrFrame);

    avcodec_free_context(&ctx);
    avformat_close_input(&formatCtx);
    return 0;
}

int main(){
    int ver = avcodec_version();
    avdevice_register_all();
    test1();
}
```