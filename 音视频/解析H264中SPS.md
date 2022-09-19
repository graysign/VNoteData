# 解析H264中SPS

```c++
#include <cstdint>
#include <cmath>

uint32_t Ue(uint8_t *pBuff, uint32_t nLen, uint32_t &nStartBit)
{
    //计算0bit的个数
    uint32_t nZeroNum = 0;
    while (nStartBit < nLen * 8)
    {
        if (pBuff[nStartBit / 8] & (0x80 >> (nStartBit % 8))) //&:按位与，%取余
        {
            break;
        }
        nZeroNum++;
        nStartBit++;
    }
    nStartBit ++;


    //计算结果
    uint32_t dwRet = 0;
    for (uint32_t i=0; i<nZeroNum; i++)
    {
        dwRet <<= 1;
        if (pBuff[nStartBit / 8] & (0x80 >> (nStartBit % 8)))
        {
            dwRet += 1;
        }
        nStartBit++;
    }
    return (1 << nZeroNum) - 1 + dwRet;
}


int Se(uint8_t *pBuff, uint32_t nLen, uint32_t &nStartBit)
{


    int UeVal=Ue(pBuff,nLen,nStartBit);
    double k=UeVal;
    int nValue=ceil(k/2);//ceil函数：ceil函数的作用是求不小于给定实数的最小整数。ceil(2)=ceil(1.2)=cei(1.5)=2.00
    if (UeVal % 2==0)
        nValue=-nValue;
    return nValue;


}


uint32_t u(uint32_t BitCount,uint8_t * buf,uint32_t &nStartBit)
{
    uint32_t dwRet = 0;
    for (uint32_t i=0; i<BitCount; i++)
    {
        dwRet <<= 1;
        if (buf[nStartBit / 8] & (0x80 >> (nStartBit % 8)))
        {
            dwRet += 1;
        }
        nStartBit++;
    }
    return dwRet;
}


bool h264_decode_seq_parameter_set(uint8_t * buf,uint32_t nLen,int &Width,int &Height)
{
    uint32_t StartBit=0;
    int forbidden_zero_bit=u(1,buf,StartBit);
    int nal_ref_idc=u(2,buf,StartBit);
    int nal_unit_type=u(5,buf,StartBit);
    if(nal_unit_type==7)
    {
        int profile_idc=u(8,buf,StartBit);
        int constraint_set0_flag=u(1,buf,StartBit);//(buf[1] & 0x80)>>7;
        int constraint_set1_flag=u(1,buf,StartBit);//(buf[1] & 0x40)>>6;
        int constraint_set2_flag=u(1,buf,StartBit);//(buf[1] & 0x20)>>5;
        int constraint_set3_flag=u(1,buf,StartBit);//(buf[1] & 0x10)>>4;
        int reserved_zero_4bits=u(4,buf,StartBit);
        int level_idc=u(8,buf,StartBit);

        int seq_parameter_set_id=Ue(buf,nLen,StartBit);

        if( profile_idc == 100 || profile_idc == 110 ||
            profile_idc == 122 || profile_idc == 144 )
        {
            int chroma_format_idc=Ue(buf,nLen,StartBit);
            if( chroma_format_idc == 3 )
                int residual_colour_transform_flag=u(1,buf,StartBit);
            int bit_depth_luma_minus8=Ue(buf,nLen,StartBit);
            int bit_depth_chroma_minus8=Ue(buf,nLen,StartBit);
            int qpprime_y_zero_transform_bypass_flag=u(1,buf,StartBit);
            int seq_scaling_matrix_present_flag=u(1,buf,StartBit);

            int seq_scaling_list_present_flag[8];
            if( seq_scaling_matrix_present_flag )
            {
                for( int i = 0; i < 8; i++ ) {
                    seq_scaling_list_present_flag[i]=u(1,buf,StartBit);
                }
            }
        }
        int log2_max_frame_num_minus4=Ue(buf,nLen,StartBit);
        int pic_order_cnt_type=Ue(buf,nLen,StartBit);
        if( pic_order_cnt_type == 0 )
            int log2_max_pic_order_cnt_lsb_minus4=Ue(buf,nLen,StartBit);
        else if( pic_order_cnt_type == 1 )
        {
            int delta_pic_order_always_zero_flag=u(1,buf,StartBit);
            int offset_for_non_ref_pic=Se(buf,nLen,StartBit);
            int offset_for_top_to_bottom_field=Se(buf,nLen,StartBit);
            int num_ref_frames_in_pic_order_cnt_cycle=Ue(buf,nLen,StartBit);

            int *offset_for_ref_frame=new int[num_ref_frames_in_pic_order_cnt_cycle];
            for( int i = 0; i < num_ref_frames_in_pic_order_cnt_cycle; i++ )
                offset_for_ref_frame[i]=Se(buf,nLen,StartBit);
            delete [] offset_for_ref_frame;
        }
        int num_ref_frames=Ue(buf,nLen,StartBit);
        int gaps_in_frame_num_value_allowed_flag=u(1,buf,StartBit);
        int pic_width_in_mbs_minus1=Ue(buf,nLen,StartBit);
        int pic_height_in_map_units_minus1=Ue(buf,nLen,StartBit);

        Width=(pic_width_in_mbs_minus1+1)*16;
        Height=(pic_height_in_map_units_minus1+1)*16;

        return true;
    }
    else
        return false;
}




int main()
{
//数据必须把H264的头0x000001去掉
//    uint8_t str[11]={0x67,0x64,0x08,0x1F,0xAC,0x34,0xC1,0x08,0x28,0x0F,0x64};
    uint8_t str[25] = {0x67,0x64,0x00,0x1F,0xAC,0x2C,0xAC,0x05,0x00,0x5B,0xB0,0x11,0x00,0x00,0x03,0x03,0xE8,0x00,0x00,0xC3,0x50,0x8F,0x1C,0x31,0x38};

    uint32_t startbit=0;
    int Width,Height;
    if (h264_decode_seq_parameter_set(str,11,Width,Height))
    {

    }

    return 0;
}
```



# 解析H265中SPS

```
#include <string.h>
struct vc_params_t{
    long width,height;
    unsigned long profile, level;
    unsigned long nal_length_size;
    void clear(){
   	    memset(this, 0, sizeof(*this));
    }
};

class NALBitstream
{
public:
    NALBitstream() : m_data(NULL), m_len(0), m_idx(0), m_bits(0), m_byte(0), m_zeros(0) {
    };

    NALBitstream(void * data, int len) {
        Init(data, len); 
    };

    void Init(void * data, int len) {
        m_data = (unsigned char*)data;
        m_len = len;
        m_idx = 0; 
        m_bits = 0;
        m_byte = 0; 
        m_zeros = 0; 
    };
    unsigned char GetBYTE(){
        if ( m_idx >= m_len )
            return 0;
        unsigned char b = m_data[m_idx++];
        if ( b == 0 ){
            m_zeros++;
            if ( (m_idx < m_len) && (m_zeros == 2) && (m_data[m_idx] == 0x03) ){
                m_idx++;
                m_zeros=0;
            }
        }else{
            m_zeros = 0;
        }
        return b;
    };
    unsigned int GetBit() {
        if (m_bits == 0) {
            m_byte = GetBYTE();
            m_bits = 8;
        }
        m_bits--;
        return (m_byte >> m_bits) & 0x1;

    };
    unsigned int GetWord(int bits) {
        unsigned int u = 0;
        while ( bits > 0 ){
            u <<= 1;
            u |= GetBit();
            bits--;
        }
        return u;
    };
	
    unsigned int GetUE() {
        int zeros = 0;
        while (m_idx < m_len && GetBit() == 0 ) 
            zeros++;
        return GetWord(zeros) + ((1 << zeros) - 1);
    };
    int GetSE(){
        unsigned int UE = GetUE();
        bool positive = UE & 1;
        int SE = (UE + 1) >> 1;
        if ( !positive )
        SE = -SE;
        return SE;
    };
private:
    unsigned char* m_data;
    int m_len;
    int m_idx;
    int m_bits;
    unsigned char m_byte;
    int m_zeros;
};


bool  ParseSequenceParameterSet(unsigned char* data,int size, vc_params_t& params){
    if (size < 20)
   	    return false;
    NALBitstream bs(data, size);
    // seq_parameter_set_rbsp()
    bs.GetWord(4);// sps_video_parameter_set_id
    int sps_max_sub_layers_minus1 = bs.GetWord(3); 
    if (sps_max_sub_layers_minus1 > 6) 
   	    return false;
    bs.GetWord(1); 
    bs.GetWord(2); 
    bs.GetWord(1); 
    params.profile = bs.GetWord(5); 
    bs.GetWord(32);//
    bs.GetWord(1);// 
    bs.GetWord(1);// 
    bs.GetWord(1);// 
    bs.GetWord(1);//  
    bs.GetWord(44);// 
    params.level   = bs.GetWord(8);// general_level_idc
    unsigned char sub_layer_profile_present_flag[6] = {0};
    unsigned char sub_layer_level_present_flag[6]   = {0};
    for (int i = 0; i < sps_max_sub_layers_minus1; i++){
        sub_layer_profile_present_flag[i]= bs.GetWord(1);
	    sub_layer_level_present_flag[i]= bs.GetWord(1);
    }


    if (sps_max_sub_layers_minus1 > 0){
        for (int i = sps_max_sub_layers_minus1; i < 8; i++) 
	        unsigned char reserved_zero_2bits = bs.GetWord(2);
    }
    for (int i = 0; i < sps_max_sub_layers_minus1; i++) {
        if (sub_layer_profile_present_flag[i]){
            bs.GetWord(2); bs.GetWord(1); bs.GetWord(5);bs.GetWord(32); bs.GetWord(1);bs.GetWord(1); bs.GetWord(1); bs.GetWord(1); bs.GetWord(44); 
        }
        if (sub_layer_level_present_flag[i])
            bs.GetWord(8);// sub_layer_level_idc[i]    
    }
    unsigned int sps_seq_parameter_set_id= bs.GetUE();  
    if (sps_seq_parameter_set_id > 15) 
        return false;
    unsigned int chroma_format_idc = bs.GetUE();  
    if (sps_seq_parameter_set_id > 3)  
        return false;
    if (chroma_format_idc == 3)  
   	    bs.GetWord(1);//  
    params.width  = bs.GetUE(); // pic_width_in_luma_samples
    params.height  = bs.GetUE(); // pic_height_in_luma_samples
    if (bs.GetWord(1)) { 
        bs.GetUE();  
        bs.GetUE();  
        bs.GetUE();  
        bs.GetUE();  
    }
    unsigned int bit_depth_luma_minus8= bs.GetUE();
    unsigned int bit_depth_chroma_minus8= bs.GetUE();
    if (bit_depth_luma_minus8 != bit_depth_chroma_minus8) 
   	    return false;
    return true;
}

int main(int argc, _TCHAR* argv[])
{
     vc_params_t params = { 0 };
     //BYTE Sps[41] = {0X42,0X01,0X01,0X01,0X60,0X00,0X00,0X03,0X00,0X80,0X00,0X00,0X03,0X00,0X00,0X03,0X00,0X5D,0XA0,0X02,0X80,0X80,0X2D,0X16,0X59,0X5E,0X49,0X32,0XB8,0X04,0X00,0X00,0X03,0X00,0X04,0X00,0X00,0X03,0X00,0X64,0X20};   
     BYTE Sps[] = { 0X42,0X01,0X01,0X01,0X60,0X00,0X00,0X03,0X00,0Xb0,0X00,0X00,0X03,0X00,0X00,0X03,0X00,0X7b,0XA0,0X03,0Xc0,0X80,0X10,0Xe5,0X8d,0Xaa,0X92,0X4c,0Xbe,0X40 };
     ParseSequenceParameterSet(Sps, 30, params);
     printf("%d-%d-%d\n", params.width, params.height, params.level);
     system("pause");   
     return 0;
 }
```