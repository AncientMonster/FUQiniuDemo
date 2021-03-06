# FUQiniuDemo

FUQiniuDemo 是 Faceunity 的面部跟踪和虚拟道具功能在 PLMediaStreamingKit 中的集成。PLMediaStreamingKit 是一个适用于 iOS 的 RTMP 直播推流 SDK，原版文档可以参考[这里](https://github.com/pili-engineering/PLMediaStreamingKit/blob/master/README.md)。

## v3.0 重要更新
在最新的版本中，全面升级了底层人脸数据库，数据库大小从原来的 10M 缩小到 3M ，同时取消了之前的 ar.mp3 数据。新的数据库可以支持稳定的全头模型，从而支持更好的道具定位、面部纹理；同时新的数据库强化了跟踪模块，从而提升虚拟化身道具的表情响应度和精度。

由于升级了底层数据表达，v2.0 版本下的道具将全面不兼容。我司制作的道具请联系我司获取升级之后的道具包。自行制作的道具请联系我司获取道具升级工具和技术支持。

v2.0 版本的系统仍然保留在 v2 分支中，但不再进行更新。

## 库文件
  - funama.h 函数调用接口头文件
  - libnama.a 人脸跟踪及道具绘制核心库    
  
## 数据文件
目录 faceunity/ 下的 \*.bundle 为程序的数据文件。数据文件中都是二进制数据，与扩展名无关。实际在app中使用时，打包在程序内或者从网络接口下载这些数据都是可行的，只要在相应的函数接口传入正确的二进制数据即可。

其中 v3.bundle 是所有道具共用的数据文件，缺少该文件会导致初始化失败。其他每一个文件对应一个道具。自定义道具制作的文档和工具请联系我司获取。
  
## 集成方法
首先把库文件拷贝到工程目录中，并添加到 xcode 工程，之后在代码中包含 funama.h 即可调用相关函数。

```C
#import "funama.h"
```

调用时，首先要创建一个openGL context:

```C
EAGLContext* g_gl_context [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
[EAGLContext setCurrentContext:g_gl_context];

```

然后对 faceunity 环境进行初始化。其中 g_auth_package 为密钥数组，没有密钥的话则传入 null 进行测试。

```C
intptr_t size = 0;
void* v3data = [self mmap_bundle:@"v3" psize:&size];
fuSetup(v3data, NULL, g_auth_package, sizeof(g_auth_package));
```

在 AVCaptureVideoDataOutputSampleBufferDelegate 的 cameraSourceDidGetPixelBuffer 回调接口中调用道具绘制函数进行绘制:

```C
static int g_frame_id = 0;
static int g_items[2] = {0, 0};

- (void *)mmap_bundle:(NSString*) fn_bundle psize:(intptr_t*)psize
{
    // Load item from predefined item bundle
    NSString *str = [[[NSBundle mainBundle] resourcePath] stringByAppendingPathComponent:[fn_bundle stringByAppendingString:@".bundle"]];
    const char *fn = [str UTF8String];
    int fd = open(fn,O_RDONLY);
    void* g_res_zip = NULL;
    size_t g_res_size = 0;
    if(fd == -1){
        NSLog(@"faceunity: failed to open bundle");
        g_res_size = 0;
    }else{
        g_res_size = [FaceUnity osal_GetFileSize:fd];
        g_res_zip = mmap(NULL, g_res_size, PROT_READ, MAP_SHARED, fd, 0);
        NSLog(@"faceunity: %@ mapped %08x %ld\n", str, (unsigned int)g_res_zip, g_res_size);
    }
    *psize = g_res_size;
    return g_res_zip;
}

- (void)captureOutput:(AVCaptureOutput *)captureOutput didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection
{
  
  if (!g_items[0])
  {
    intptr_t size = 0;    
    void* data = [self mmap_bundle:@"XXXX.bundle" psize:&size];    
    // key item creation function call
        g_items[0] = fuCreateItemFromPackage(data, (int)size);
  }

    CVPixelBufferRef pixelBuffer = CMSampleBufferGetImageBuffer(sampleBuffer);    
    CVPixelBufferLockBaseAddress(pixelBuffer, 0);    
    int h = (int)CVPixelBufferGetHeight(pixelBuffer);        
    int stride = (int)CVPixelBufferGetBytesPerRow(pixelBuffer);        
    int* img = (int*)CVPixelBufferGetBaseAddress(pixelBuffer);        
    fuRenderItems(0, img, stride/4, h, g_frame_id, g_items, 1);        
    CVPixelBufferUnlockBaseAddress(pixelBuffer, 0);    
    g_frame_id++;
}
```

## 视频美颜
美颜功能实现步骤与道具类似，首先加载美颜道具，并将fuCreateItemFromPackage返回的值赋给g_items[1]:
  
```C
g_items[1] = fuCreateItemFromPackage(g_res_zip, (int)g_res_size);
```

在调用fuRenderItems()前设置美颜相关参数:

```C
//  Set item parameters
fuItemSetParamd(g_items[1], "color_level", 1.0);
fuItemSetParams(g_items[1], "filter_name", g_filter_names[g_selected_filter]);
fuItemSetParamd(g_items[1], "blur_radius", 8.0);
```

其中fuItemSetParamd为传参函数，可通过此函数来设置美颜程度、滤镜种类等相关参数。其中level为设置美颜程度参数，范围在0～1之间，数值越大美颜程度越强。filter_name为滤镜名称，主要包括下方这几种，可通过传入不同的滤镜名称来切换滤镜种类。这里需要注意的是：默认使用"nature"作为美白滤镜，而不在使用"none"作为默认滤镜。

```C
"nature", "delta", "electric", "slowlived", "tokyo", "warm"
```

在滤镜为 `nature` 时，参数 `color_level` 控制美白的程度，其值为 1.0 时为默认美白的程度，大于 1.0 的参数值可以进一步强化美白效果。该参数也对其他滤镜有效，其设置方法如下：

```C
fuItemSetParamd(g_items[1], "color_level", 1.0);
```

参数 `blur_radius` 控制美颜磨皮的程度，数值为磨皮滤波的半径。中等磨皮可以设置为 8.0 ，重度磨皮可以设置为 16.0:

```C
fuItemSetParamd(g_items[1], "blur_radius", 8.0);
```

最后，在调用fuRenderItems()接口时需要注意接口的最后一个参数。如果只开了道具或美颜，请将此参数设为1，如果同时开启了美颜和道具功能，请将此值设为2.

```C
fuRenderItems(0, img, stride/4, h, g_frame_id, g_items, 2);
```

## OC封装层

在原有SDK基础上对fuSetup及fuRenderItemsEx这两个函数进行了封装，总共包括三个接口：

- fuSetup接口封装

```C
+ (void)setupWithData:(void *)data ardata:(void *)ardata authPackage:(void *)package authSize:(int)size;

```
- 单输入接口：输入一个pixelBuffer并返回一个加过美颜或道具的pixelBuffer，支持YUV及BGRA格式出入，且输出与输入格式一致。

```C
- (CVPixelBufferRef)renderPixelBuffer:(CVPixelBufferRef)pixelBuffer withFrameId:(int)frameid items:(int*)items itemCount:(int)itemCount;
```
- 双输入接口：输入pixelBuffer及texture，然后返回一个FUOutput结构体，结构体中包含的就是加过美颜或道具的pixelBuffer及texture。输入的pixelBuffer支持YUV及BGRA格式，输入的texture只支持BGRA格式，输出与输入格式一致。

```C
- (FUOutput)renderPixelBuffer:(CVPixelBufferRef)pixelBuffer bgraTexture:(GLuint)textureHandle withFrameId:(int)frameid items:(int *)items itemCount:(int)itemCount;

```
- FUOutput结构体：包含一个pixelBuffer和一个texture

```C
typedef struct{
    CVPixelBufferRef pixelBuffer;
    GLuint bgraTextureHandle;
}FUOutput;

```

## 鉴权

我们的系统通过标准TLS证书进行鉴权。客户在使用时先从发证机构申请证书，之后将证书数据写在客户端代码中，客户端运行时发回我司服务器进行验证。在证书有效期内，可以正常使用库函数所提供的各种功能。没有证书或者证书失效等鉴权失败的情况会限制库函数的功能，在开始运行一段时间后自动终止。

证书类型分为**两种**，分别为**发证机构证书**和**终端用户证书**。

#### - 发证机构证书
**适用对象**：此类证书适合需批量生成终端证书的机构或公司，比如软件代理商，大客户等。

发证机构的二级CA证书必须由我司颁发，具体流程如下。

1. 机构生成私钥
机构调用以下命令在本地生成私钥 CERT_NAME.key ，其中 CERT_NAME 为机构名称。
```
openssl ecparam -name prime256v1 -genkey -out CERT_NAME.key
```

2. 机构根据私钥生成证书签发请求
机构根据本地生成的私钥，调用以下命令生成证书签发请求 CERT_NAME.csr 。在生成证书签发请求的过程中注意在 Common Name 字段中填写机构的正式名称。
```
openssl req -new -sha256 -key CERT_NAME.key -out CERT_NAME.csr
```

3. 将证书签发请求发回我司颁发机构证书

之后发证机构就可以独立进行终端用户的证书发行工作，不再需要我司的配合。

如果需要在终端用户证书有效期内终止证书，可以由机构自行用OpenSSL吊销，然后生成pem格式的吊销列表文件发给我们。例如如果要吊销先前误发的 "bad_client.crt"，可以如下操作：
```
openssl ca -config ca.conf -revoke bad_client.crt -keyfile CERT_NAME.key -cert CERT_NAME.crt
openssl ca -config ca.conf -gencrl -keyfile CERT_NAME.key -cert CERT_NAME.crt -out CERT_NAME.crl.pem
```
然后将生成的 CERT_NAME.crl.pem 发回给我司。

#### - 终端用户证书
**适用对象**：直接的终端证书使用者。比如，直接客户或个人等。

终端用户由我司或者其他发证机构颁发证书，并通过我司的证书工具生成一个代码头文件交给用户。该文件中是一个常量数组，内容是加密之后的证书数据，形式如下。
```
static char g_auth_package[]={ ... }
```

用户在库环境初始化时，需要提供该数组进行鉴权，具体参考 fuSetup 接口。没有证书、证书失效、网络连接失败等情况下，会造成鉴权失败，在控制台或者Android平台的log里面打出 "not authenticated" 信息，并在运行一段时间后停止渲染道具。

任何其他关于授权问题，请email：support@faceunity.com

## 函数接口及参数说明
### [3.0.2] - 2016-12-28
1. 增强人脸识别稳定性

```
## 函数接口及参数说明

```C
/**
\brief Initialize and authenticate your SDK instance to the FaceUnity server, must be called exactly once before all other functions.
  The buffers should NEVER be freed while the other functions are still being called.
  You can call this function multiple times to "switch pointers".
\param v2data should point to contents of the "v2.bin" we provide
\param ardata should point to contents of the "ar.bin" we provide
\param authdata is the pointer to the authentication data pack we provide. You must avoid storing the data in a file.
  Normally you can just `#include "authpack.h"` and put `g_auth_package` here.
\param sz_authdata is the authentication data size, we use plain int to avoid cross-language compilation issues.
  Normally you can just `#include "authpack.h"` and put `sizeof(g_auth_package)` here.
*/
void fuSetup(float* v2data,float* ardata,void* authdata,int sz_authdata);

/**
\brief Call this function when the GLES context has been lost and recreated.
  That isn't a normal thing, so this function could leak resources on each call.
*/
void fuOnDeviceLost();

/**
\brief Call this function to reset the face tracker on camera switches
*/
void fuOnCameraChange();

/**
\brief Create an accessory item from a binary package, you can discard the data after the call.
  This function MUST be called in the same GLES context / thread as fuRenderItems.
\param data is the pointer to the data
\param sz is the data size, we use plain int to avoid cross-language compilation issues
\return an integer handle representing the item
*/
int fuCreateItemFromPackage(void* data,int sz);

/**
\brief Destroy an accessory item.
  This function MUST be called in the same GLES context / thread as the original fuCreateItemFromPackage.
\param item is the handle to be destroyed
*/
void fuDestroyItem(int item);

/**
\brief Destroy all accessory items ever created.
  This function MUST be called in the same GLES context / thread as the original fuCreateItemFromPackage.
*/
void fuDestroyAllItems();

/**
\brief Render a list of items on top of a GLES texture or a memory buffer.
  This function needs a GLES 2.0+ context.
\param texid specifies a GLES texture. Set it to 0u if you want to render to a memory buffer.
\param img specifies a memory buffer. Set it to NULL if you want to render to a texture.
  If img is non-NULL, it will be overwritten by the rendered image when fuRenderItems returns
\param w specifies the image width
\param h specifies the image height
\param frameid specifies the current frame id. 
  To get animated effects, please increase frame_id by 1 whenever you call this.
\param p_items points to the list of items
\param n_items is the number of items
\return a new GLES texture containing the rendered image in the texture mode
*/
int fuRenderItems(int texid,int* img,int w,int h,int frame_id, int* p_items,int n_items);

/*\brief An I/O format where `ptr` points to a BGRA buffer. It matches the camera format on iOS. */
#define FU_FORMAT_BGRA_BUFFER 0
/*\brief An I/O format where `ptr` points to a single GLuint that is a RGBA texture. It matches the hardware encoding format on Android. */
#define FU_FORMAT_RGBA_TEXTURE 1
/*\brief An I/O format where `ptr` points to an NV21 buffer. It matches the camera preview format on Android. */
#define FU_FORMAT_NV21_BUFFER 2
/*\brief An output-only format where `ptr` is NULL. The result is directly rendered onto the current GL framebuffer. */
#define FU_FORMAT_GL_CURRENT_FRAMEBUFFER 3
/*\brief An I/O format where `ptr` points to a RGBA buffer. */
#define FU_FORMAT_RGBA_BUFFER 4

/**
\brief Generalized interface for rendering a list of items.
  This function needs a GLES 2.0+ context.
\param out_format is the output format
\param out_ptr receives the rendering result, which is either a GLuint texture handle or a memory buffer
\param in_format is the input format
\param in_ptr points to the input image, which is either a GLuint texture handle or a memory buffer
\param w specifies the image width
\param h specifies the image height
\param frameid specifies the current frame id. 
  To get animated effects, please increase frame_id by 1 whenever you call this.
\param p_items points to the list of items
\param n_items is the number of items
\return a GLuint texture handle containing the rendering result if out_format isn't FU_FORMAT_GL_CURRENT_FRAMEBUFFER
*/
int fuRenderItemsEx(
  int out_format,void* out_ptr,
  int in_format,void* in_ptr,
  int w,int h,int frame_id, int* p_items,int n_items);
  
/**
\brief Set an item parameter to a double value
\param item specifies the item
\param name is the parameter name
\param value is the parameter value to be set
\return zero for failure, non-zero for success
*/
int fuItemSetParamd(int item,char* name,double value);

/**
\brief Set an item parameter to a double array
\param item specifies the item
\param name is the parameter name
\param value points to an array of doubles
\param n specifies the number of elements in value
\return zero for failure, non-zero for success
*/
int fuItemSetParamdv(int item,char* name,double* value,int n);

/**
\brief Set an item parameter to a string value
\param item specifies the item
\param name is the parameter name
\param value is the parameter value to be set
\return zero for failure, non-zero for success
*/
int fuItemSetParams(int item,char* name,char* value);

/**
\brief Get an item parameter to a double value
\param item specifies the item
\param name is the parameter name
\return double value of the parameter
*/
double fuItemGetParamd(int item,char* name);

/**
\brief Get the face tracking status
\return zero for not tracking, non-zero for tracking
*/
int fuIsTracking();

/**
\brief Set the default orientation for face detection. The correct orientation would make the initial detection much faster.
One of 0..3 should work.
*/
void fuSetDefaultOrientation(int rmode);
```
