###### javax.imageio.ImageIO
```bash
#JDk版本
java version "1.6.0_02"
Java(TM) SE Runtime Environment (build 1.6.0_02-b06)
Java HotSpot(TM) Client VM (build 1.6.0_02-b06, mixed mode, sharing)
```

ImageIO可以处理的图像类型  
```java
System.out.println(Arrays.asList(ImageIO.getReaderFormatNames()));  
```
```bash
输出：[jpg, BMP, bmp, JPG, wbmp, jpeg, png, PNG, JPEG, WBMP, GIF, gif]
#看可以看到JDK1.6通过ImageIO可以加载以上类型的图片。
```

###### 通过BufferedImage来处理图片
```java
BufferedImage bi=ImageIO.read(new File("D:/Test/binary/test.jpg"));//通过imageio将图像载入  
int h=bi.getHeight();//获取图像的高  
int w=bi.getWidth();//获取图像的宽  
int rgb=bi.getRGB(0, 0);//获取指定坐标的ARGB的像素值  
```
可以看出通过BufferedImage 就可以轻松的获取图像的尺寸。  
`int BufferedImage.getRGB(x,y).`获取到的是ARGB，最前面多了一个透明度。将返回的int转为16进制后,0-1透明度，2-3是R，4-5是G，6-7是B。  

###### 具体二值化的算法

* 首先获取每个像素点的灰度值。目前使用简单的(R+G+B)/3
* 然后选定一个阀值。
* 将一个像素点灰度值和它周围的8个灰度值相加再除以9，然后和阀值比较。大于阀值的则置为255，小于则是0  


###### 实现
```java
package image.binary;  
  
import java.awt.Color;  
import java.awt.image.BufferedImage;  
import java.io.File;  
import java.io.IOException;  
  
import javax.imageio.ImageIO;  
  
public class BinaryTest {  
      
    public static void main(String[] args) throws IOException {  
        BufferedImage bi=ImageIO.read(new File("D:/Test/binary/test.jpg"));//通过imageio将图像载入  
        int h=bi.getHeight();//获取图像的高  
        int w=bi.getWidth();//获取图像的宽  
        int rgb=bi.getRGB(0, 0);//获取指定坐标的ARGB的像素值  
        int[][] gray=new int[w][h];  
        for (int x = 0; x < w; x++) {  
            for (int y = 0; y < h; y++) {  
                gray[x][y]=getGray(bi.getRGB(x, y));  
            }  
        }  
          
        BufferedImage nbi=new BufferedImage(w,h,BufferedImage.TYPE_BYTE_BINARY);  
        int SW=160;  
        for (int x = 0; x < w; x++) {  
            for (int y = 0; y < h; y++) {  
                if(getAverageColor(gray, x, y, w, h)>SW){  
                    int max=new Color(255,255,255).getRGB();  
                    nbi.setRGB(x, y, max);  
                }else{  
                    int min=new Color(0,0,0).getRGB();  
                    nbi.setRGB(x, y, min);  
                }  
            }  
        }  
          
        ImageIO.write(nbi, "jpg", new File("D:/Test/binary/二值化后_无压缩.jpg"));  
    }  
  
    public static int getGray(int rgb){  
        String str=Integer.toHexString(rgb);  
        int r=Integer.parseInt(str.substring(2,4),16);  
        int g=Integer.parseInt(str.substring(4,6),16);  
        int b=Integer.parseInt(str.substring(6,8),16);  
        //or 直接new个color对象  
        Color c=new Color(rgb);  
        r=c.getRed();  
            g=c.getGreen();  
        b=c.getBlue();  
        int top=(r+g+b)/3;  
        return (int)(top);  
    }  
      
    /** 
     * 自己加周围8个灰度值再除以9，算出其相对灰度值 
     * @param gray 
     * @param x 
     * @param y 
     * @param w 
     * @param h 
     * @return 
     */  
    public static int  getAverageColor(int[][] gray, int x, int y, int w, int h)  
    {  
        int rs = gray[x][y]  
                        + (x == 0 ? 255 : gray[x - 1][y])  
                        + (x == 0 || y == 0 ? 255 : gray[x - 1][y - 1])  
                        + (x == 0 || y == h - 1 ? 255 : gray[x - 1][y + 1])  
                        + (y == 0 ? 255 : gray[x][y - 1])  
                        + (y == h - 1 ? 255 : gray[x][y + 1])  
                        + (x == w - 1 ? 255 : gray[x + 1][ y])  
                        + (x == w - 1 || y == 0 ? 255 : gray[x + 1][y - 1])  
                        + (x == w - 1 || y == h - 1 ? 255 : gray[x + 1][y + 1]);  
        return rs / 9;  
    }  
  
}  
```
相对较简单，将一个jpg图片二值化后产生一个无压缩的jpg文件。  


###### 压缩  
有些时候是需要将二值化后的图片进行压缩的，然后转成tiff格式的文件，以减少存储成本。但是jdk1.6的imageIO还没有加入TIFF格式的图像处理，那么可以使用辅助的包了。  

* jai-imageio包。目前发布的最新版本为1.1,1.2还在开发中。。`http://download.java.net/media/jai-imageio/builds/release/1_0_01/jai_imageio-1_0_01-windows-i586-jar.zip`
* 解压后就两个jar包和一个dll。dll丢进PATH中，jar包放入工程，则我们就可以做压缩。

```java
System.out.println(Arrays.asList(ImageIO.getWriterFormatNames()));  
```
看看，加入jar后的Imageio：
```bash
[raw, tif, jpeg, JFIF, WBMP, jpeg-lossless, jpeg-ls, PNM, JPG, wbmp, PNG, JPEG, jpeg 2000, tiff, BMP, JPEG2000, RAW, JPEG-LOSSLESS, jpeg2000, GIF, TIF, TIFF, bmp, jpg, pnm, png, jfif, JPEG 2000, gif, JPEG-LS]
```
是不是可以处理的格式更强大了呢。  

* 压缩的全部代码

```java
package image.binary;  
  
import java.awt.Color;  
import java.awt.image.BufferedImage;  
import java.io.File;  
import java.io.IOException;  
  
import javax.imageio.IIOImage;  
import javax.imageio.ImageIO;  
import javax.imageio.ImageWriteParam;  
import javax.imageio.ImageWriter;  
import javax.imageio.stream.ImageOutputStream;  
  
import com.sun.media.imageioimpl.plugins.tiff.TIFFImageWriterSpi;  
  
public class BinaryTest {  
      
    public static void main(String[] args) throws IOException {  
        BufferedImage bi=ImageIO.read(new File("D:/Test/binary/test.jpg"));//通过imageio将图像载入  
        int h=bi.getHeight();//获取图像的高  
        int w=bi.getWidth();//获取图像的宽  
        int[][] gray=new int[w][h];  
        for (int x = 0; x < w; x++) {  
            for (int y = 0; y < h; y++) {  
                gray[x][y]=getGray(bi.getRGB(x, y));  
            }  
        }  
          
        BufferedImage nbi=new BufferedImage(w,h,BufferedImage.TYPE_BYTE_BINARY);  
        int SW=160;  
        for (int x = 0; x < w; x++) {  
            for (int y = 0; y < h; y++) {  
                if(getAverageColor(gray, x, y, w, h)>SW){  
                    int max=new Color(255,255,255).getRGB();  
                    nbi.setRGB(x, y, max);  
                }else{  
                    int min=new Color(0,0,0).getRGB();  
                    nbi.setRGB(x, y, min);  
                }  
            }  
        }  
          
        TIFFImageWriterSpi tiffws=new TIFFImageWriterSpi();  
        ImageWriter writer=tiffws.createWriterInstance();  
        ImageWriteParam param=writer.getDefaultWriteParam();  
        param.setCompressionMode(ImageWriteParam.MODE_EXPLICIT);  
        param.setCompressionType("CCITT T.6");  
        param.setCompressionQuality(0.8f);  
        File outFile=new File("D:/Test/binary/二值化后_有压缩.tiff");  
        ImageOutputStream ios=ImageIO.createImageOutputStream(outFile);  
        writer.setOutput(ios);  
        writer.write(null,new IIOImage(nbi, null, null), param);  
    }  
  
    public static int getGray(int rgb){  
        String str=Integer.toHexString(rgb);  
        int r=Integer.parseInt(str.substring(2,4),16);  
        int g=Integer.parseInt(str.substring(4,6),16);  
        int b=Integer.parseInt(str.substring(6,8),16);  
        //or 直接new个color对象  
        Color c=new Color(rgb);  
        r=c.getRed();  
        g=c.getGreen();  
        b=c.getBlue();  
        int top=(r+g+b)/3;  
        return (int)(top);  
    }  
      
    /** 
     * 自己加周围8个灰度值再除以9，算出其相对灰度值 
     * @param gray 
     * @param x 
     * @param y 
     * @param w 
     * @param h 
     * @return 
     */  
    public static int  getAverageColor(int[][] gray, int x, int y, int w, int h)  
    {  
        int rs = gray[x][y]  
                        + (x == 0 ? 255 : gray[x - 1][y])  
                        + (x == 0 || y == 0 ? 255 : gray[x - 1][y - 1])  
                        + (x == 0 || y == h - 1 ? 255 : gray[x - 1][y + 1])  
                        + (y == 0 ? 255 : gray[x][y - 1])  
                        + (y == h - 1 ? 255 : gray[x][y + 1])  
                        + (x == w - 1 ? 255 : gray[x + 1][ y])  
                        + (x == w - 1 || y == 0 ? 255 : gray[x + 1][y - 1])  
                        + (x == w - 1 || y == h - 1 ? 255 : gray[x + 1][y + 1]);  
        return rs / 9;  
    }  
  
}  
```