+++
title = '有关favorite的立绘合成'
date = 2024-04-11T07:45:07+08:00

+++

首先我们需要知道favorite的立绘通常在一个叫做`graph_bs.bin`的文件下，用`grabro`打开可以看到如下内容：

![1](1.png)

![2](2.png)

很明显，主体部分和面部表情是分开的，这种情况我们称之为差分

而`garbro`并不会自动处理差分内容，所以如果要提取人物立绘就得我们自己手动合成

那么如何实现呢，我们先使用`010editor`打开这个文件看看结构

![3](3.png)

看起来。。。好像什么都看不出来

不用着急，我们先看看最简单的`bgm.bin`来认识一下`*.bin`的文件结构

先用使用`garbro`打开，可以看到，这个`bgm.bin`文件下共有60个文件

![4](4.png)

使用`010editor`打开我们可以发现地址0h-4h刚好也等于60

![5](5.png)

接着是一个值大小为240，这个值我们先不用理他

![6](6.png)

现在我们先使用`garbro`随便提取一个文件，与原`bgm.bin`文件进行比较

![7](7.png)

你会发现，好家伙原来这文件数据就在`.bin`文件里面，他甚至不愿意异或一下

起始地址为968，共2723547个字节

接着我们返回原`bgm.bin`一看，哎这不是968和2723547吗·

![8](8.png)

![9](9.png)

于是我们对前面这一部分就有了一点点了解，但是这里每个文件的第一个数据每次加4，这个4是拿来干什么的的呢？不着急，我们慢慢看。

![10](10.png)

接着，当我们读完这些数据后，会发现之后这个东西

![11](11.png)

wow，这不是文件名吗，还有原来之前的240是指的文件名的长度，那不难猜出前面的4指的就是文件名的大小的，也就是第一块是文件的文件名在文件名块（index）里面的地址

![12](12.png)

接着打开`graph_bs.bin`验证我们的猜想

![13](13.png)

完全符合，虽然第一个不是0但也对应了第一个文件的文件名所在的地址

![14](14.png)

我们甚至可以知道文件名是以00分割的

这下我们不难写出提取文件的程序了，但问题来了，我们提取出来的是一个`.hzc`文件啊，相当于`garbro`提取时的保留原样

![15](15.png)

`ffmpeg`都无法正确识别这个格式

![16](16.png)

但`garbro`居然可以正确识别，~~这个时候我们就应该去翻看`garbro`的源码了~~

```C#
//! \file       ArcHZC.cs
//! \date       Wed Dec 09 17:04:23 2015
//! \brief      Favorite View Point multi-frame image format.
//
// Copyright (C) 2015 by morkt
//
// Permission is hereby granted, free of charge, to any person obtaining a copy
// of this software and associated documentation files (the "Software"), to
// deal in the Software without restriction, including without limitation the
// rights to use, copy, modify, merge, publish, distribute, sublicense, and/or
// sell copies of the Software, and to permit persons to whom the Software is
// furnished to do so, subject to the following conditions:
//
// The above copyright notice and this permission notice shall be included in
// all copies or substantial portions of the Software.
//
// THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
// IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
// FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
// AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
// LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
// FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
// IN THE SOFTWARE.
//

using System;
using System.Collections.Generic;
using System.ComponentModel.Composition;
using System.IO;
using GameRes.Utility;
using GameRes.Compression;

namespace GameRes.Formats.FVP
{
    internal class HzcArchive : ArcFile
    {
        public readonly HzcMetaData ImageInfo;

        public HzcArchive (ArcView arc, ArchiveFormat impl, ICollection<Entry> dir, HzcMetaData info)
            : base (arc, impl, dir)
        {
            ImageInfo = info;
        }
    }

    [Export(typeof(ArchiveFormat))]
    public class HzcOpener : ArchiveFormat
    {
        public override string         Tag { get { return "HZC/MULTI"; } }
        public override string Description { get { return "Favorite View Point multi-frame image"; } }
        public override uint     Signature { get { return 0x31637A68; } } // 'HZC1'
        public override bool  IsHierarchic { get { return false; } }
        public override bool      CanWrite { get { return false; } }

        public HzcOpener ()
        {
            Extensions = new string[] { "hzc" };
        }

        static readonly Lazy<ImageFormat> Hzc = new Lazy<ImageFormat> (() => ImageFormat.FindByTag ("HZC"));

        public override ArcFile TryOpen (ArcView file)
        {
            uint header_size = file.View.ReadUInt32 (8);
            HzcMetaData image_info;
            using (var header = file.CreateStream (0, 0xC+header_size))
            {
                image_info = Hzc.Value.ReadMetaData (header) as HzcMetaData;
                if (null == image_info)
                    return null;
            }
            int count = file.View.ReadInt32 (0x20);
            if (0 == count)
                count = 1;
            string base_name = Path.GetFileNameWithoutExtension (file.Name);
            int frame_size = image_info.UnpackedSize / count;
            var dir = new List<Entry> (count);
            for (int i = 0; i < count; ++i)
            {
                var entry = new Entry {
                    Name = string.Format ("{0}#{1:D3}", base_name, i),
                    Type = "image",
                    Offset = frame_size * i,
                    Size = (uint)frame_size,
                };
                dir.Add (entry);
            }
            return new HzcArchive (file, this, dir, image_info);
        }

        public override Stream OpenEntry (ArcFile arc, Entry entry)
        {
            var hzc = (HzcArchive)arc;
            using (var input = arc.File.CreateStream (0xC+hzc.ImageInfo.HeaderSize))
            using (var z = new ZLibStream (input, CompressionMode.Decompress))
            {
                uint frame_size = entry.Size;
                var pixels = new byte[frame_size];
                uint offset = 0;
                for (;;)
                {
                    if (pixels.Length != z.Read (pixels, 0, pixels.Length))
                        throw new EndOfStreamException();
                    if (offset >= entry.Offset)
                        break;
                    offset += frame_size;
                }
                return new BinMemoryStream (pixels, entry.Name);
            }
        }

        public override IImageDecoder OpenImage (ArcFile arc, Entry entry)
        {
            var hzc = (HzcArchive)arc;
            var input = arc.File.CreateStream (0xC+hzc.ImageInfo.HeaderSize);
            try
            {
                return new HzcDecoder (input, hzc.ImageInfo, entry);
            }
            catch
            {
                input.Dispose();
                throw;
            }
        }
    }
}
```

不难看出，等等？ZLibStream？

![17](17.png)

这下看懂了，原来图像数据是使用`zlib`进行压缩了

![18](18.png)

有关mode：

0代表背景为`rgb`，1代表立绘基底为`rgba`，2代表差分（多个文件）也为`rgba`

接着那串0000之后就是`zlib`压缩的数据了，解压之后就可以得到`rgba/rgb`

![19](19.png)

然后利用`opencv`直接合成就好

实现代码如下（~~写的好屎，勿喷~~）：

GitHub： https://github.com/bGlzdGRlcg/unfavbs

`unfavbs.h`

```c++
#include <bits/stdc++.h>
#include <iostream>
#include <fstream>
#include <vector>
#include <opencv2/opencv.hpp>
#include <sstream>
#include <zlib.h>
#include <windows.h>
#include <locale>
#include <codecvt>
#include <map>
#include <io.h>

using namespace std;
using namespace cv;

unsigned int readbinhelf(ifstream &binfile){
	unsigned short int val;
	binfile.read((char *) &val, 2);
	return val;
}

unsigned int readbin(ifstream &binfile){
	unsigned int val;
	binfile.read((char *) &val, 4);
	return val;
}

unsigned int readpng(istringstream &pngfile){
	unsigned int val;
	pngfile.read((char *) &val, 4);
	return val;
}

unsigned int readpnghelf(istringstream &pngfile){
	unsigned short int val;
	pngfile.read((char *) &val, 2);
	return val;
}

string ShiftJISToUTF8(const string& shiftjis) {
    int len = MultiByteToWideChar(932,0,shiftjis.c_str(),-1,NULL,0);
    if (len == 0) return "";
    wstring wstr(len,0);
    MultiByteToWideChar(932,0,shiftjis.c_str(),-1,&wstr[0],len);
    len = WideCharToMultiByte(CP_UTF8,0,&wstr[0],-1,NULL,0,NULL,NULL);
    if (len == 0) return "";
    std::string utf8(len,0);
    WideCharToMultiByte(CP_UTF8,0,&wstr[0],-1,&utf8[0],len,NULL,NULL);
    return utf8;
}

string ShiftJISToGBK(const string& shiftjis) {
    int len = MultiByteToWideChar(932, 0, shiftjis.c_str(), -1, NULL, 0);
    if (len == 0) return "";
    std::wstring wstr(len, 0);
    MultiByteToWideChar(932, 0, shiftjis.c_str(), -1, &wstr[0], len);
    len = WideCharToMultiByte(936, 0, wstr.c_str(), -1, NULL, 0, NULL, NULL);
    if (len == 0) return "";
    string gbk(len, 0);
    WideCharToMultiByte(936, 0, wstr.c_str(), -1, &gbk[0], len, NULL, NULL);
    return gbk;
}

string getbinname(ifstream &binfile){
	string str="";
	char ch;
    while (binfile.get(ch)) {
        if (int(ch) == 0) break;
        str += ch;
    }
    return ShiftJISToGBK(str);
}

bool cmp(int a[],int b[]) {
    return a[0] < b[0];
}

void unbinout(ifstream &binfile,ofstream &outfile, streampos start, streampos end) {
    binfile.seekg(start);
    char buffer[1024];
    streampos remaining = end - start;
    while (remaining > 0) {
        size_t bytesToRead = min(remaining, static_cast<streampos>(sizeof(buffer)));
        binfile.read(buffer, bytesToRead);
        outfile.write(buffer, bytesToRead);
        remaining -= bytesToRead;
    }
    outfile.close();
}

istringstream decompressData(ifstream &inputFile, int startOffset, int uncompressedSize) {
    inputFile.seekg(startOffset);
    vector<char> buffer(uncompressedSize);
    inputFile.read(buffer.data(), uncompressedSize);
    vector<char> uncompressedBuffer(uncompressedSize * 2);
    uLongf destLen = uncompressedSize * 2;
    int result = uncompress((Bytef*)uncompressedBuffer.data(), &destLen, (const Bytef*)buffer.data(), uncompressedSize);
    if(result != Z_OK) {
        cerr<<"Failed to decompress data. Error code: " <<result<<endl;
        return istringstream();
        //return "";
    }
    string decompressedData(uncompressedBuffer.begin(), uncompressedBuffer.begin() + destLen);
    //return decompressedData;
	istringstream iss(decompressedData);
    return iss;
}

void overlayImages(const Mat &background, const Mat &foreground, Mat &output, int x, int y){
    output = background.clone();
    cv::Rect roi(x, y, foreground.cols, foreground.rows);
    roi &= cv::Rect(0, 0, background.cols, background.rows);
    cv::Mat roi_output = output(roi);
    cv::Mat roi_foreground = foreground(cv::Rect(0, 0, roi.width, roi.height));
    for (int i = 0; i < roi.height; ++i){
        for (int j = 0; j < roi.width; ++j){
            cv::Vec4b pixel_foreground = roi_foreground.at<cv::Vec4b>(i, j);
            cv::Vec4b &pixel_output = roi_output.at<cv::Vec4b>(i, j);
            double alpha_foreground = pixel_foreground[3] / 255.0;
            double alpha_background = 1.0 - alpha_foreground;
            for (int k = 0; k < 3; ++k){
                pixel_output[k] = static_cast<uchar>(alpha_foreground * pixel_foreground[k] + alpha_background * pixel_output[k]);
            }
            pixel_output[3] = static_cast<uchar>((alpha_foreground * 255) + (alpha_background * pixel_output[3]));
        }
    }
}

```

`unfavbs.cpp`

```cpp
#include "unfavbs.h"

//定义导出文件数量，以及文件名在bin文件内所占字节大小
int filenamesize,filenumber;

int main(int argc,char* argv[]){
	
	//创建output文件夹
	system("md graph_bs");
	system("cls");

	//打开graph_bs.bin文件
	ifstream binfile;
	binfile.open("graph_bs.bin",ios::in | ios::binary);
	
	//从bin文件读入变量filenumber和filenamesize
	filenumber = readbin(binfile);
	filenamesize = readbin(binfile);
	//cout<<filenumber<<" "<<filenamesize<<endl;
	
	/*
		初始化动态数组bin和unbinname
		bin用来存放对应序列号，文件大小以及起始地址
		unbinname用来存放输出的文件名
	*/
	int **bin = new int *[filenumber];
	string *unbinname = new string[filenumber];
	for(int i = 0;i < filenumber;i++) bin[i] = new int[3];
	
	//从bin文件内读入bin数组
	for(int i = 0;i < filenumber;i++) for(int j = 0;j < 3;j++) bin[i][j] = readbin(binfile);
	
	//排序，使得bin可以与unbinname对应
	sort(bin,bin+filenumber,cmp);
	
	//读入unbinname
	for(int i = 0;i < filenumber;i++) unbinname[i] = getbinname(binfile);
	for(int i = 0;i < filenumber;i++) unbinname[i].erase(unbinname[i].length()-1);
	//for(int i = 0;i < filenumber;i++) cout<<bin[i][0]<<" "<<bin[i][1]<<" "<<bin[i][2]<<" "<<unbinname[i]<<" "<<unbinname[i].size()<<endl;

	//定义一个vecoter用来保存表情文件对应下标
	vector<int>face;

	//保存表情文件下标
	//-79 -19 -57 -23为GBK字符: "表情"
	for(int i = 0;i < filenumber;i++) 
		if(int(unbinname[i][unbinname[i].size()-4])==-79 &&
		int(unbinname[i][unbinname[i].size()-3])==-19 &&
		int(unbinname[i][unbinname[i].size()-2])==-57 &&
		int(unbinname[i][unbinname[i].size()-1])==-23)
			face.push_back(i);
			
	//创建一个map映射文件名-->下标
	map<string,int>binmap;
	for(int i = 0;i < filenumber;i++) binmap[unbinname[i]]=i;

	//定义一个vecoter用来保存每个表情的基底
	vector<int>base;

	//计算每个基底的下标
	for(int i=0;i<face.size();i++){
		string temp="";
		for(int t=0;t<unbinname[face[i]].size()-5;t++) temp+=unbinname[face[i]][t];
		if(unbinname[binmap[temp]]==temp) base.push_back(binmap[temp]);
		else {
			for(int t=i;t<face.size()-1;t++) swap(face[t],face[t+1]);
			face.pop_back();
			i--;
		}
	}
	//for(int i=0;i<face.size();i++) cout<<unbinname[face[i]]<<" "<<unbinname[base[i]]<<" "<<face[i]<<endl;

	//清空之前的缓存
	system("del cachebase");
	system("del cacheface");
	system("cls");
	
	//合成立绘并输出到graph_bs文件夹
	for(int i=0;i<base.size();i++){
		//定义两个流basestream和facestream
		ifstream basestream,facestream;

		//定义临时文件流
		ofstream cachebase,cacheface;
		cachebase.open("cachebase",ios::out | ios::binary);
		cacheface.open("cacheface",ios::out | ios::binary);

		//读入basestream
		unbinout(binfile,cachebase,bin[base[i]][1],bin[base[i]][2]+bin[base[i]][1]);
		basestream.open("cachebase",ios::in | ios::binary);

		//读入facestream
		unbinout(binfile,cacheface,bin[face[i]][1],bin[face[i]][2]+bin[face[i]][1]);
		facestream.open("cacheface",ios::in | ios::binary);

		//读入底图解压后的数据大小
		readbin(basestream);
		int basepngsize=readbin(basestream);

		//读取底图的画布大小
		readbin(basestream);
		readbin(basestream);
		readbin(basestream);
		int basepngx=readbinhelf(basestream),basepngy=readbinhelf(basestream);
		readbin(basestream);
		readbin(basestream);
		readbin(basestream);
		readbin(basestream);
		readbin(basestream);
		istringstream basepngstream=decompressData(basestream,44,basepngsize);

		//创建底图
		Mat basepng(basepngy,basepngx, CV_8UC4);
		for(int j = 0; j < basepngy; ++j) for(int k = 0; k < basepngx; ++k) {
        	basepng.at<cv::Vec4b>(j, k)[0] = int(basepngstream.get());
        	basepng.at<cv::Vec4b>(j, k)[1] = int(basepngstream.get());
        	basepng.at<cv::Vec4b>(j, k)[2] = int(basepngstream.get());
        	basepng.at<cv::Vec4b>(j, k)[3] = int(basepngstream.get());
    	}

		//读入表情解压后的数据大小
		readbin(facestream);
		int facepngsize=readbin(facestream);
		readbin(facestream);
		readbin(facestream);
		readbin(facestream);

		//读表情的画布大小以及偏移坐标
		int facepngx=readbinhelf(facestream),facepngy=readbinhelf(facestream),x=readbinhelf(facestream),y=readbinhelf(facestream);
		readbin(facestream);
		
		//读入表情图片数量
		int pngnumber=readbin(facestream);
		readbin(facestream);
		readbin(facestream);
		istringstream facepngstream=decompressData(facestream,44,facepngsize);

		//合成并输出
		string cmdstr="md .\\graph_bs\\"+unbinname[base[i]];
    	system(cmdstr.c_str());
    	system("cls");
    	for(int n=0;n<pngnumber;n++) {
    		Mat facepng(facepngy,facepngx, CV_8UC4);
    		for(int k = 0; k < facepngy; ++k) for(int j = 0; j < facepngx; ++j) {
        		facepng.at<cv::Vec4b>(k, j)[0] = int(facepngstream.get());
        		facepng.at<cv::Vec4b>(k, j)[1] = int(facepngstream.get());
        		facepng.at<cv::Vec4b>(k, j)[2] = int(facepngstream.get());
        		facepng.at<cv::Vec4b>(k, j)[3] = int(facepngstream.get());
    		}
			Mat pngfile;
			overlayImages(basepng,facepng,pngfile,x,y);
    		imwrite(".\\graph_bs\\"+unbinname[base[i]]+'\\'+unbinname[base[i]]+'_'+to_string(n)+".png",pngfile);
		}
		basestream.close();
		facestream.close();
		system("del cachebase");
		system("del cacheface");
		system("cls");
	}

	//释放动态数组
	for(int i = 0;i < filenumber;i++) delete[] bin[i];
	delete[] bin;
	delete[] unbinname;
	
	//关闭bin文件
	binfile.close();
	
	return 0;
}

```

