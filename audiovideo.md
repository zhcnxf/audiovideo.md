# 视频
## 视频压缩／转码
### VideoCompressor
VideoCompressor使用安卓4.0以上系统提供的MediaCodec做MP4视频文件的解码／编码(硬件编／解码)。VideoCompressor的核心逻辑如下图：

                                                                                                    
               (original.mp4)                                                                       
                     |                                                                              
                     |                        seek to zero                                          
           [MediaExtractor/Video]      - - - - - - - - - - - - - ->  [MediaExtractor/Audio]         
                     |                                                          |                   
                     |                                                          |                   
                     | Sample data &                                            |                   
                     | presentation time                                        |                   
                     |                                                          |                   
                     |                                                          |                   
                     |                                                          |                   
            [MediaCodec/Decoder] --- [Surface] --- [SurfaceTexture]             |                   
                                                           |                    |                   
                                                           |                    |                   
                                                      [GL Texture]              |                   
                                                           |                    |                   
                                                           |                    |                   
                                        rotate/transform   |                    |                   
            [TextureRenderer(GLES20)] ---------------------|                    | Sample data &     
                                                           |                    | presentation time 
                                                           |                    |                   
                                                     [EGL Display]              |                   
                                                           |                    |                   
                                                           |                    |                   
            [MediaCodec/Encoder] ------------- [Surface(Input)]                 |                   
                     |                                                          |                   
                     |                                                          |                   
                     |                                                          |                   
                     |                                                          |                   
                     |                                                          |                   
                     |    Video track                          Audio track      |                   
                     └-------------------- [MediaMuxer] ------------------------┘                   
                                                  |                                                 
                                                  |                                                 
                                           (compressed.mp4)                                                            

#### MediaExtractor
MediaExtractor类提供了视频文件分轨道读取音视频帧数据和元信息的功能。
```java
    MediaExtractor extractor = new MediaExtractor();
    extractor.setDataSource(mp4FilePath); // 设置视频文件

    MediaFormat mediaFormat = extractor.getTrackFormat(trackIndex); // 读取轨道格式

    extractor.selectTrack(videoTrack); // 选中轨道

    ByteBufer byteBuffer = ...;
    int readSize = extractor.readSampleData(byteBuffer, offset); // 读取轨道数据到buffer

    ...
    //# 定位帧
	extractor.seekTo(timeUs, MediaExtractor.SEEK_TO_PREVIOUS_SYNC); // 定位至某一时间
    extractor.advance(); // 定位至下一帧
```

#### MediaCodec
MediaCodec是安卓提供的硬件编码／解码视频文件的类。MediaCodec类通过Surface来作为编码输入和解码输出。

##### 解码
```java
    //# 创建解码器
    String mime = videoFormat.getString(MediaFormat.KEY_MIME); // 轨道格式MIME
    MediaCodec decoder = MediaCodec.createDecoderByType(mime); // 创建视频解码器
	SurfaceTexture texture = new SurfaceTexture(textureId); // 创建Texture, 这里的textureId是由GLES20创建纹理得到的纹理ID
    Surface outputSurface = new Surface(texture);  // 创建解码输出Surface
    decoder.configure(videoFormat, surface, null, 0); // 配置解码器，指定解码输出Surface

    ...

    //# 给解码器输送数据
 
    // 从decoder获取用来填数据的ByteBuffer
    int timeout = 1000;
	int bufferIndex = decoder.dequeueInputBuffer(timeout);
    ByteBuffer sampleBuffer = decoder.getInputBuffer(bufferIndex);    

    // MediaExtractor读取视频轨道数据到sampleBuffer
    int chunkSize = extractor.readSampleData(sampleBuffer, offset);    

    // 放回sampleBuffer, 完成数据输送
    decoder.queueInputBuffer(bufferIndex, 0, chunkSize, extractor.getSampleTime(), 0);
```

##### 编码
```java
    //# 创建编码器
    MediaCodec encoder = MediaCodec.createEncoderByType("video/avc");  // 创建视频编码器对象
    encoder.configure(outputFormat, null, null, MediaCodec.CONFIGURE_FLAG_ENCODE); // 配置编码器，指定编码视频格式
    Surface inputSurface = encoder.createInputSurface(); // 获得编码器输入Surface, 在这个Surface上渲染的视图将被编码为视频帧

    ...

    //# 读取编码数据

    // 
    int timeout = 1000;
    MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
    int bufferIndex = encoder.dequeueOutputBuffer(bufferInfo, timeout); 
    ByteBuffer outputBuffer = encoder.getOutputBuffer(bufferIndex);

    // 这里outputBuffer是编码过的数据，一般这里把它发送给复用器来存储到视频轨道

    // 最后回收encodedBuffer
    encoder.releaseOutputBuffer(bufferIndex, false);
```

#### 视频帧处理
EGL是连接Android framework和GLES的框架。GLES可以通过EGLSurface对象和EGLDisplay对象来把变换后的纹理图像提交给编码器的Surface。
```java
    EGLDisplay eglDisplay = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);

    //# decoder视频帧到GLES纹理
    // GLES创建纹理
    int[] glTextures = new int[1];
	GLES20.glGenTextures(1, glTextures, 0);
    int textureId = glTextures[0];

    // 基于GLES纹理创建SurfaceTexture
    SurfaceTexture texture = new SurfaceTexture(textureId);

    // 这个outpuSurface最为decoder的输出Surface
    Surface outputSurface = new Surface(texture); 

    // 绑定纹理，GLES就能通过textureId来处理decoder解码出的视频帧
    GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, textureId);
    

    //# 编码变换后的视频帧

    // 基于encoder的输入Surface创建EGLSurface
    EGLSurface eglSurface =  EGL14.eglCreateWindowSurface(eglDisplay, configs[0], inputSurface, surfaceAttribs, 0); // 这里inputSurface就是encoder的输入Surface
    
    // 提交EGLDisplay数据到encoder的输入Surface
    GLES20.glSwapBuffer(eglDisplay, eglSurface);
    
    // 至此decoder解码出的视频帧处理过后交给了encoder去编码
```

#### MP4Builder
使用MP4Builder对象把编码的视频轨道数据和音频轨道数据合成后保存成MP4文件。
```java
    // 创建MP4Builder对象
    Mp4Movie movie = new Mp4Movie();
    movie.setCacheFile(cacheFile); // 指定保存的MP4文件
    movie.setRotation(rotationValue);
    movie.setSize(resultWidth, resultHeight);
    MP4Builder mediaMuxer = new MP4Builder().createMovie(movie);

    // 添加媒体轨道
    int videoTrack = mediaMuxer.addTrack(videoFormat, false);
    int audioTrack = mediaMuxer.addTrack(audioFormat, false);

    // 写入媒体数据
    mediaMuxer.writeSampleData(videoTrack, encodedBuffer, info, false);
```

### Ffmpeg
jni_ffmpeg提供了处理视频文件的命令式接口，如 ffmpeg -i original.mp4 [参数列表] compressed.mp4，Ffmpeg命令使用可以参考 https://www.ffmpeg.org/ffmpeg.html

# 音频
## 录制声音
### 功能和权限
```xml
    <uses-feature android:name="android.hardware.microphone" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
```

### AudioRecord对象
AudioRecord对象提供了从音频输入设备读取声音数据的功能。

#### 选择采样率

由于不同设备支持的声音采样率不同(所有设备都支持44100)，可以用以下方法找到设备支持的采样率
```java
    for (int rate : new int[] {8000, 11025, 16000, 22050, 44100}) {
        int bufferSize = AudioRecord.getMinBufferSize(rate, AudioFormat.CHANNEL_CONFIGURATION_DEFAULT, AudioFormat.ENCODING_PCM_16BIT);
        if (bufferSize > 0) {
            // buffer size is valid, Sample rate supported
        }
    }
```

#### 创建AndioRecord对象
```java
    int audioSource = MediaRecorder.AudioSource.MIC; // 输入设备麦克风
    int sampleRateInHz = 44100; // 44100或者其他设备支持的采样率
    int audioFormat = AudioFormat.ENCODING_PCM_16BIT; // 声音数据格式，16位深
    int channelConfig = AudioFormat.CHANNEL_IN_MONO; // 单声道
    int bufferSizeInShorts = AudioRecord.getMinBufferSize(sampleRateInHz, channelConfig, AudioFormat.ENCODING_PCM_16BIT);
    int bufferSizeInBytes = bufferSizeInShorts * 2;
    AudioRecord audioRecord = new AudioRecord(audioSource, sampleRateInHz, channelConfig, audioFormat, bufferSizeInBytes);
```

#### 读取声音数据
读取声音数据的方法
```java
    int read (short[] audioData, int offsetInShorts, int sizeInShorts,  int readMode)     // 对应AudioFormat.ENCODING_PCM_16BIT

    int read (float[] audioData, int offsetInFloats, int sizeInFloats, int readMode)    // 对应AudioFormat.ENCODING_PCM_FLOAT

    int read (byte[] audioData, int offsetInBytes, int sizeInBytes, int readMode)   // 对应AudioFormat.ENCODING_PCM_8BIT

    int read (ByteBuffer audioBuffer, int sizeInBytes, int readMode)

    // readMode：0 blocking, 1 non-blocking
```

```java
    AudioRecord audioRecord = ...

    short[] rawBuffer = new short[bufferSizeInShorts];

    audioRecord.startRecording();

    while(!needStop) {
        int readSize = audioRecord.read(rawBuffer, 0, rawBuffer.length); // Read blocking
        // ...
    }
 
    audioRecord.stop();
    audioRecord.release();
```

### LAME
使用开源项目mp3lame编码器。mp3lame把原始声音数据编码成MP3数据帧。

     ┌─---──-──┐                      ┌---────-─┐
     │ PCM     |       mp3lame        | MP3     |      append
     │ data    |  ─ ─ ─ ─ ─ - ─ ─ >   | frame   |  -------------> (file.mp3)
     |         |                      |         |
     └──-─----─┘                      └-──----──┘  

#### SimpleLame类
SimpleLame类封装了lame jni库，提供几个简单的方法：
```java
    //  初始化lame
	public native static void init(int inSamplerate, int outChannel, int outSamplerate, int outBitrate, int quality /** 压缩度 0..9 **/);

    // 编码数据帧
	public native static int encode(short[] buffer_l, short[] buffer_r, int samples, byte[] mp3buf);

    // Flush lame buffer
    public native static int flush(byte[] mp3buf);

    // 关闭lame，释放buffer
    public native static void close();
```

#### 录制MP3文件
```java
    AudioRecord audioRecord = ...

    short[] rawBuffer = new short[bufferSizeInShorts];
    byte[] mp3buffer = new byte[(int) (7200 + rawBuffer.length * 2 * 1.25)]; // A large enough buffer according to mp3lame docs.

    // 开始录制
    audioRecord.startRecording();

    // 初始化lame
    SimpleLame.init(sampleRate, 1, sampleRate, 32);

    while(!needStop) {
         // blocking read raw data
        int readSize = audioRecord.read(rawBuffer, 0, rawBuffer.length);

        // encode mp3
        int encResult = SimpleLame.encode(rawBuffer, rawBuffer, readSize, mp3Buffer);
        if (encResult < 0) {
            // error handling
        } else {
            // 写入文件
            fileOutputStream.write(mp3Buffer, 0, encResult);
        }
    }

    // 冲刷mp3lame buffer
    int flushResult = SimpleLame.flush(mp3Buffer);
    if (flushResult < 0) {
        // error handling
    } else if (flushResult != 0) {
        fileOutputStream.write(mp3Buffer, 0, flushResult);
    }

    // 释放资源
    fileOutputStream.close();
    SimpleLame.close();
    audioRecord.stop();
    audioRecord.release();
```

# 文件缓存

     ┌─---──---┐                   ┌---────-─┐
     │ Remote  |       http        |         |      http
     │ file    |  - - - - - - ->   |  Proxy  |  ------------>  (player)
     |         |                   |         |
     └──-─----─┘                   └-───┬--──┘  
                                        | file io
                                    ┌──-┴─-─┐
                                    │ Local |
                                    │ file  |
                                    └────--─┘
                                        