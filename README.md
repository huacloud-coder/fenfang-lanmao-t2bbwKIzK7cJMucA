
本文主要介绍摄像头（相机）如何采集数据，用于类似摄像头本地显示软件，以及流媒体数据传输场景如传屏、视讯会议等。


摄像头采集有多种方案，如AForge.NET、WPFMediaKit、OpenCvSharp、EmguCv、DirectShow.NET、MediaCaptre（UWP），网上一些文章以及github已经有很多介绍，这里总结、确认技术选型给大家一个参考


### 1\. AForge.NET


AForge视频库是基于DirectShow技术开发的，提供了捕捉、处理和显示视频流接口，以及图像丰富的图像处理功能，如滤镜、特征提取和物体检测。详见官网开源仓库 [andrewkirillov/AForge.NET(github.com)](https://github.com)


我们下面看下AForge录制代码，安装Nuget包依赖：




```
1     "AForge.Video" Version="2.2.5" />
2     "AForge.Video.DirectShow" Version="2.2.5" />
3     "System.Drawing.Common" Version="8.0.8" />
```


 摄像头显示：




```
 1     private void StartButton_OnClick(object sender, RoutedEventArgs e)
 2     {
 3         // 获取所有视频输入设备
 4         var videoDevices = new FilterInfoCollection(FilterCategory.VideoInputDevice);
 5         if (videoDevices.Count > 0)
 6         {
 7             // 选择第一个视频输入设备
 8             var videoSource = new VideoCaptureDevice(videoDevices[0].MonikerString);
 9             // 注册NewFrame事件处理程序
10             videoSource.NewFrame += new NewFrameEventHandler(videoSource_NewFrame);
11             // 开始摄像头视频源
12             videoSource.Start();
13         }
14     }
15 
16     private async void videoSource_NewFrame(object sender, NewFrameEventArgs eventArgs)
17     {
18         // 获取当前的视频帧，显示
19         var image = ToBitmapImage(eventArgs.Frame);
20         await Dispatcher.InvokeAsync(() => { CaptureImage.Source = image; });
21     }
```


摄像头录制视频流：




```
 1     private async void videoSource_NewFrame1(object sender, NewFrameEventArgs eventArgs)
 2     {
 3         // 获取当前的视频帧
 4         // 将Bitmap转换为byte[]，用于流媒体传输
 5         byte[] byteArray = BitmapToByteArray(eventArgs.Frame, out int stride);
 6         // 将byte[]转换为BitmapImage，用于临时展示
 7         BitmapImage image = ByteArrayToBitmapImage(byteArray, eventArgs.Frame.Width, eventArgs.Frame.Height, stride, eventArgs.Frame.PixelFormat);
 8         await Dispatcher.InvokeAsync(() => { CaptureImage.Source = image; });
 9     }
```


其中的数据转换，这里要把分辨率stride同byte\[]数据一同储存，不然后续数据是无法处理的：




```
 1     // 将Bitmap转换为byte[]
 2     public byte[] BitmapToByteArray(Bitmap bitmap, out int stride)
 3     {
 4         Rectangle rect = new Rectangle(0, 0, bitmap.Width, bitmap.Height);
 5         BitmapData bitmapData = bitmap.LockBits(rect, ImageLockMode.ReadOnly, bitmap.PixelFormat);
 6         //stride是分辨率水平值，如3840
 7         stride = bitmapData.Stride;
 8         int bytes = Math.Abs(bitmapData.Stride) * bitmap.Height;
 9         byte[] rgbValues = new byte[bytes];
10 
11         // 复制位图数据到字节数组
12         Marshal.Copy(bitmapData.Scan0, rgbValues, 0, bytes);
13 
14         bitmap.UnlockBits(bitmapData);
15         return rgbValues;
16     }
17 
18     // 将byte[]转换为BitmapImage
19     public BitmapImage ByteArrayToBitmapImage(byte[] byteArray, int width, int height, int stride, PixelFormat pixelFormat)
20     {
21         var bitmapImage = new BitmapImage();
22         using (var memoryStream = new MemoryStream())
23         {
24             var bmp = new Bitmap(width, height, stride, pixelFormat, Marshal.UnsafeAddrOfPinnedArrayElement(byteArray, 0));
25             // 保存到MemoryStream中
26             bmp.Save(memoryStream, System.Drawing.Imaging.ImageFormat.Bmp);
27             memoryStream.Seek(0, SeekOrigin.Begin);
28             bitmapImage.BeginInit();
29             bitmapImage.StreamSource = memoryStream;
30             bitmapImage.CacheOption = BitmapCacheOption.OnLoad;
31             bitmapImage.EndInit();
32             bitmapImage.Freeze();
33         }
34         return bitmapImage;
35     }
```


详见Demo代码 [kybs00/AForgeNETDemo (github.com)](https://github.com)


经验证，**延迟较大，对比Windows系统相机不够清晰**


### 2\. WPFMediaKit


WPFMediaKit也是基于DirectShow的，它提供了一些封装便于在WPF应用中使用媒体功能。[Sascha\-L/WPF\-MediaKit (github.com)](https://github.com)


使用 WPFMediaKit 要录制摄像头视频，需要结合 WPFMediaKit 提供的视频捕获功能和其他库（例如 AForge 或 FFmpeg）来实现录制功能。


这里引用WPFMediaKit、AForge.Video.FFMPEG俩个Nuget包，然后通过定时器捕获当前视频帧：




```
 1     var bitmap = new Bitmap(videoCaptureElement.Width, videoCaptureElement.Height, System.Drawing.Imaging.PixelFormat.Format24bppRgb);
 2     var bitmapData = bitmap.LockBits(new Rectangle(0, 0, bitmap.Width, bitmap.Height),System.Drawing.Imaging.ImageLockMode.WriteOnly,bitmap.PixelFormat);
 3     try
 4     {
 5         videoCaptureElement.VideoCaptureDevice.GetCurrentVideoFrame(out IntPtr frame);
 6         System.Runtime.InteropServices.Marshal.Copy(frame, 0, bitmapData.Scan0, videoCaptureElement.Width * videoCaptureElement.Height * 3);
 7     }
 8     finally
 9     {
10         bitmap.UnlockBits(bitmapData);
11     }
```


这个定时器的实现比较low。**延时较低、较流畅，但与Win系统相机对比也是不够清晰**


### 3\.MediaCapture（UWP）


MediaCapture是Windows 8及以上版本的WinRT API，专为捕获音频、视频和照片设计。


MediaCaptuer是UWP应用的API [使用 MediaCapture 捕获基本的照片、视频和音频 \- UWP applications \| Microsoft Learn](https://github.com)，要在WPF内使用需要引入俩个Nuget包：




```
1     "Microsoft.Toolkit.Wpf.UI.XamlHost" Version="6.1.2" />
2     "Microsoft.Windows.SDK.Contracts" Version="10.0.26100.1" />
```


初始化MediaCapture:




```
1     var mediaCapture =new MediaCapture();
2     var videos = await DeviceInformation.FindAllAsync(DeviceClass.VideoCapture);
3     var settings = new MediaCaptureInitializationSettings()
4     {
5         VideoDeviceId = videos[0].Id,
6         StreamingCaptureMode = StreamingCaptureMode.Video,
7     };
8     await mediaCapture.InitializeAsync(settings);
```


分几个场景，分别输出Demo。录制本地文件，注释已经很详细了并不重复解释，直接看代码：




```
 1     private MediaCapture _mediaCapture;
 2     private InMemoryRandomAccessStream _randomAccessStream;
 3     private async void StartButton_OnClick(object sender, RoutedEventArgs e)
 4     {
 5         // 1. 初始化 MediaCapture 对象
 6         var mediaCapture = _mediaCapture = new MediaCapture();
 7         var videos = await DeviceInformation.FindAllAsync(DeviceClass.VideoCapture);
 8         var settings = new MediaCaptureInitializationSettings()
 9         {
10             VideoDeviceId = videos[0].Id,
11             StreamingCaptureMode = StreamingCaptureMode.Video,
12         };
13         await mediaCapture.InitializeAsync(settings);
14 
15         // 2. 设置要录制的数据流
16         var randomAccessStream = _randomAccessStream = new InMemoryRandomAccessStream();
17         // 3. 配置录制的视频设置
18         var mediaEncodingProfile = MediaEncodingProfile.CreateMp4(VideoEncodingQuality.Auto);
19         // 4. 开始录制
20         await mediaCapture.StartRecordToStreamAsync(mediaEncodingProfile, randomAccessStream);
21     }
22 
23     private async void StopButton_OnClick(object sender, RoutedEventArgs e)
24     {
25         // 停止录制
26         await _mediaCapture.StopRecordAsync();
27         // 处理录制后的数据,保存至"C:\Users\XXX\Videos\RecordedVideo.mp4"
28         var storageFolder = Windows.Storage.KnownFolders.VideosLibrary;
29         var file = await storageFolder.CreateFileAsync("RecordedVideo.mp4", Windows.Storage.CreationCollisionOption.GenerateUniqueName);
30         using var fileStream = await file.OpenAsync(Windows.Storage.FileAccessMode.ReadWrite);
31         await RandomAccessStream.CopyAndCloseAsync(_randomAccessStream.GetInputStreamAt(0), fileStream.GetOutputStreamAt(0));
32         _randomAccessStream.Dispose();
33     }
```


摄像头显示，通过UWP\-WindowsXamlHost承载画面（置顶）：




```
 1     private async void StartButton_OnClick(object sender, RoutedEventArgs e)
 2     {
 3         _mediaCapture = new MediaCapture();
 4         var videos = await DeviceInformation.FindAllAsync(DeviceClass.VideoCapture);
 5         var settings = new MediaCaptureInitializationSettings()
 6         {
 7             VideoDeviceId = videos[0].Id,
 8             StreamingCaptureMode = StreamingCaptureMode.Video,
 9         };
10         await _mediaCapture.InitializeAsync(settings);
11         //显示WindowsXamlHost
12         VideoViewHost.Visibility = Visibility.Visible;
13         //绑定画面源
14         _captureElement.Source = _mediaCapture;
15         await _mediaCapture.StartPreviewAsync();
16     }
```


MediaCapture是UWP平台的实现方案，直接给CaptureElement赋值绑定画面源。直接用CaptureElement渲染速度很快，这个实现逻辑同windows系统相机是一样的


另外，使用MediaCapture也可以捕获画面帧事件，用于流媒体数据捕获收集：




```
1 // 配置视频帧读取器
2 var frameSource = mediaCapture.FrameSources.Values.FirstOrDefault(source => source.Info.MediaStreamType == MediaStreamType.VideoRecord);
3 _frameReader = await mediaCapture.CreateFrameReaderAsync(frameSource, MediaEncodingSubtypes.Argb32);
4 _frameReader.FrameArrived += FrameReader_FrameArrived;
5 await _frameReader.StartAsync();
```


如下方所示，监听FrameArrived，使用Windows.UI.Xaml.Media.Imaging.BitmapImage渲染展示（仅用于展示，延迟很高）：




```
 1     private async void FrameReader_FrameArrived(MediaFrameReader sender, MediaFrameArrivedEventArgs args)
 2     {
 3         var frame = sender.TryAcquireLatestFrame();
 4         if (frame != null)
 5         {
 6             var bitmap = frame.VideoMediaFrame?.SoftwareBitmap;
 7             if (bitmap != null)
 8             {
 9                 // 在这里对每一帧进行处理
10                 await Dispatcher.InvokeAsync(async () =>
11                 {
12                     var bitmapImage = await ConvertSoftwareBitmapToBitmapImageAsync(bitmap);
13                     _captureImage.Source = bitmapImage;
14                 });
15             }
16         }
17     }
```


如需要将SoftwareBitmap转为buffer字节数据，可以按如下处理：




```
 1     public async Task<byte[]> SoftwareBitmapToByteArrayAsync(SoftwareBitmap softwareBitmap)
 2     {
 3         // 使用InMemoryRandomAccessStream来存储图像数据
 4         using var stream = new InMemoryRandomAccessStream();
 5         // 创建位图编码器
 6         var encoder = await BitmapEncoder.CreateAsync(BitmapEncoder.PngEncoderId, stream);
 7         // 转换为BGRA8格式，如果当前格式不同
 8         var bitmap = SoftwareBitmap.Convert(softwareBitmap, BitmapPixelFormat.Bgra8, BitmapAlphaMode.Premultiplied);
 9         encoder.SetSoftwareBitmap(bitmap);
10         await encoder.FlushAsync();
11         bitmap.Dispose();
12 
13         // 读取字节数据
14         using var reader = new DataReader(stream.GetInputStreamAt(0));
15         byte[] byteArray = new byte[stream.Size];
16         await reader.LoadAsync((uint)stream.Size);
17         reader.ReadBytes(byteArray);
18 
19         return byteArray;
20     }
```


以上，MediaCapture能实现摄像头显示及录制相关功能。MediaCapture代码Demo详见Github：[kybs00/CameraCaptureDemo: 摄像头预览、捕获DEMO (github.com)](https://github.com)


### 3\.其它


OpenCvSharp是OpenCV在C\#环境中的包装，提供跨平台的计算机视觉和图像处理功能 [shimat/opencvsharp: OpenCV wrapper for .NET (github.com)](https://github.com)。**2K视频较为流畅，4K视频延迟较低但显示效果较差**


其它的如EmguCV是另一个基于OpenCV的C\#包装组件库，具有与OpenCVSharp相同的强大功能 [emgucv/emgucv: Emgu CV (github.com)](https://github.com)


DirectShow.NET，提供对DirectShow API的托管包装，使得在.NET框架中可以直接使用DirectShow的强大功能来进行视频捕获和处理 [pauldotknopf/DirectShow.NET](https://github.com):[westworld加速](https://tianchuang88.com)。DirectShow本身性能较好，但DirectShow.NET作为托管包装，性能会受一定影响。延迟效果待验证


此处就不一一例写Demo了


我们看看性能数据，4K屏设备\+4K摄像头，通过本地摄像头预览显示。借用组内小伙伴建凯大佬对各方案的延时统计数据：


![](https://img2024.cnblogs.com/blog/685541/202408/685541-20240829174939765-753455281.png)


验证了下MediaCaptre的延时与系统相机差不多。通过任务管理器我们可以看到，系统相机CPU占用3%，但GPU是15%。


![](https://img2024.cnblogs.com/blog/685541/202408/685541-20240829205452893-489022041.png)


使用了硬件加速，性能方面很不错，所以摄像头采集推荐MediaCaptre方案


值得一提的是，公司大屏HDMI采集卡信号Hdmi Record即6911龙讯固件，采用MediaCapture采集画面会稳定很多，减少了黑屏、粉屏的概率。从这点也说明Windows系统相机的原生实现方案，兼容性更好


另外，这里验证的方案都是针对4K摄像头，如果是8K摄像头，其性能要求更高了，后面单独介绍


 


参考列表：


[使用 MediaCapture 捕获基本的照片、视频和音频 \- UWP applications \| Microsoft Learn](https://github.com)


[\[C\#] 使用Accord.Net，实现相机画面采集，视频保存及裁剪视频区域，利用WriteableBitmap高效渲染 \- 孤独成派 \- 博客园 (cnblogs.com)](https://github.com)


[.net中捕获摄像头视频的方式及对比(How to Capture Camera Video via .Net) \- Wuya \- 博客园 (cnblogs.com)](https://github.com)


[C\# 利用AForge进行摄像头信息采集 \- 老码识途呀 \- 博客园 (cnblogs.com)](https://github.com)


[WPF摄像头使用(WPFMediaKit) \- 深秋无痕 \- 博客园 (cnblogs.com)](https://github.com)


[C\#使用OpenCvSharp4库读取电脑摄像头数据并实时显示\_c\#显示摄像头画面\-CSDN博客](https://github.com)


[SharpCaptureDemo: C\#，vb, .net采集摄像头，桌面屏幕，麦克风话筒声音，摄像头画面，并且支持混音，也支持同时采集录制。效率高，底层采用了directshow技术，稳定性强，兼容性好。 (gitee.com)](https://github.com)


