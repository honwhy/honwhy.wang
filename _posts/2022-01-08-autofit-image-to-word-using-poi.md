---
layout: post
category: java
comments: true
title: 将图片自适应大小插入到word文档
tags: collect
---
* content
{:toc}

## 概述
如果手工新建word文档，插入图片是可以做到图片自适应等比缩放的，那么如何通过程序的方式来完成呢。

## demo
通过一番搜索，了解到Java操作word添加图片的方法，[Adding Images to a Word Document using Java](https://www.geeksforgeeks.org/adding-images-to-a-word-document-using-java/);
重要的就是这几步
```
Step 1: Creating a blank document
Step 2: Creating a Paragraph
Step 3: Creating a File output stream of word document at the required location
Step 4: Creating a file input stream of the image by specifying its path
Step 5: Retrieving the image file name and image type
Step 6: Setting the width and height of the image in pixels
Step 7: Adding the picture using the addPicture() method and writing into the document
Step 8: Closing the connections
```
其中添加图片的重要代码如下，
```java
XWPFParagraph paragraph
            = document.createParagraph();
XWPFRun run = paragraph.createRun();

// Step 3: Creating a File output stream of word
// document at the required location
FileOutputStream fout = new FileOutputStream(
	new File("D:\\WordFile.docx"));

// Step 4: Creating a file input stream of image by
// specifying its path
File image = new File("D:\\Images\\image.jpg");
FileInputStream imageData
	= new FileInputStream(image);

// Step 5: Retrieving the image file name and image
// type
int imageType = XWPFDocument.PICTURE_TYPE_JPEG;
String imageFileName = image.getName();

// Step 6: Setting the width and height of the image
// in pixels.
int width = 450;
int height = 400;

// Step 7: Adding the picture using the addPicture()
// method and writing into the document
run.addPicture(imageData, imageType, imageFileName,
			   Units.toEMU(width),
			   Units.toEMU(height));
```
首先创建一个段落，然后XWPFRun辅助类来添加图片，添加图片需要知道图片的像素大小。

# 优化1
关于图片大小的获取，demo代码中并没有处理，应该这么操作，
```java
        BufferedImage img = ImageIO.read(image);
        int width = img.getWidth();
        int height = img.getHeight()
```
# 优化2
重要的是图片缩放计算，首先要直到常规A4纸张的大小，A4纸内容区域的大小。
A4纸宽=210mm，高=297mm，页面的内容左右边距分别都是31.8mm，上下边距分别是25.4mm；
只有这些尺寸是不够的，还不清楚这些尺寸可以保存多少像素。


通过搜索发现，A4纸在屏幕上的像素尺寸信息，https://cloud.tencent.com/developer/article/1505088；
分辨率是96像素/英寸时，A4纸的尺寸的图像的像素是794×1123；(默认)，而1英寸=2.54cm。

这句话的意思是，如果一个图片的宽度是1280px那么换算成尺寸应该是，1280 / 96.0 * 25.4 = 330.2mm

# 优化3
直到了怎么处理图片像素到尺寸的转换，那么就可以考虑如何处理等比例缩放问题了。
1、计算图片宽带与内容区域宽度的比例
2、计算图片高度与内容区域高度的比例
3、取两个比例之中最小的值，如果比例大于1则图片不缩放保持原来的宽度和高度，否则执行缩放
```java
BufferedImage img = ImageIO.read(image);
int width = img.getWidth();
int height = img.getHeight();
// image size
double mw = width / 96.0 * 25.4;
double mh = height / 96.0 * 25.4;
// content size
double cw = 210 - (2*31.8);
double ch = 297 - (2*25.4);

double scaling = 1.0;
if (mw > cw) {
    scaling = cw / mw;
}
if (mh > ch) {
    if (ch / mh < scaling) {
        scaling = ch / mh;
    }
}
```

# 汇总
```java
    // add image
    private static void addPicture(File image, XWPFDocument document, FileOutputStream fout) throws IOException, InvalidFormatException {
        FileInputStream imageData = new FileInputStream(image);
        // Step 2: Creating a Paragraph using
        // createParagraph() method
        XWPFParagraph paragraph
                = document.createParagraph();
        XWPFRun run = paragraph.createRun();
        //run.addCarriageReturn();
        // Step 5: Retrieving the image file name and image
        // type
        int imageType = XWPFDocument.PICTURE_TYPE_JPEG;
        String imageFileName = image.getName();

        // Step 6: Setting the width and height of the image
        // in pixels.
        BufferedImage img = ImageIO.read(image);
        int width = img.getWidth();
        int height = img.getHeight();
        // image size
        double mw = width / 96.0 * 25.4;
        double mh = height / 96.0 * 25.4;
        // content size
        double cw = 210 - (2*31.8);
        double ch = 297 - (2*25.4);

        double scaling = 1.0;
        if (mw > cw) {
            scaling = cw / mw;
        }
        if (mh > ch) {
            if (ch / mh < scaling) {
                scaling = ch / mh;
            }
        }
        // Step 7: Adding the picture using the addPicture()
        // method and writing into the document
        run.addPicture(imageData, imageType, imageFileName,
                Units.pixelToEMU((int) (width * scaling * .9)),
                Units.pixelToEMU((int) (height * scaling * .9)));
    }
```
# 完整的代码
```java
    public static void main(String[] args) throws IOException, InvalidFormatException {


        // Step 4: Creating a file input stream of image by
        // specifying its path
        String year = "2021";
        File[] months = FileUtil.ls("d:/data/" + year);
        // Step 1: Creating a blank document
        XWPFDocument document = new XWPFDocument();
        // Step 3: Creating a File output stream of word
        // document at the required location
        FileOutputStream fout = new FileOutputStream(
                new File("D:\\WordFile-" + year + ".docx"));
        // months
        for (File month : months) {
            // dates
            File[] dates = FileUtil.ls(month.getAbsolutePath());
            for (File date : dates) {
                // images
                File[] images = FileUtil.ls(date.getAbsolutePath());
                for (File image : images) {
                    String mimeType = FileUtil.getMimeType(image.getAbsolutePath());
                    if ("image/jpeg".equals(mimeType)) {
                        addPicture(image, document, fout);
                    } else {

                    }
                }
            }
        }
        document.write(fout);
        // Step 8: Closing the connections
        fout.close();
        document.close();

    }
```
重要的依赖
```xml
  <dependencies>
    <dependency>
      <groupId>cn.hutool</groupId>
      <artifactId>hutool-all</artifactId>
      <version>5.7.18</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.poi/poi -->
    <dependency>
      <groupId>org.apache.poi</groupId>
      <artifactId>poi</artifactId>
      <version>5.1.0</version>
    </dependency>

    <!-- https://mvnrepository.com/artifact/org.apache.poi/poi-ooxml -->
    <dependency>
      <groupId>org.apache.poi</groupId>
      <artifactId>poi-ooxml</artifactId>
      <version>5.1.0</version>
    </dependency>
    <!-- https://mvnrepository.com/artifact/org.apache.poi/poi-ooxml-schemas -->
    <dependency>
      <groupId>org.apache.poi</groupId>
      <artifactId>poi-ooxml-schemas</artifactId>
      <version>4.1.2</version>
    </dependency>
```