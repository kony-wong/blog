
[toc]

## AlsaAudio


```
 % aplay -vv somefile.wav
 % amixer scontrols
```

### 构成

```
    The currently designed interfaces are listed below:
    Information Interface (/proc/asound)
    Control Interface (/dev/snd/controlCX)
    Mixer Interface (/dev/snd/mixerCXDX)
    PCM Interface (/dev/snd/pcmCXDX)
    Raw MIDI Interface (/dev/snd/midiCXDX)
    Sequencer Interface (/dev/snd/seq)
    Timer Interface (/dev/snd/timer)
```


以上是alsa driver提供的对外接口。
对上的接口，alsa通过alsa-lib来提供面向应用层的接口，避免直接操作这些驱动接口

## 安卓内的音频系统

参考 Android_Media_Framework.md，目前在作成中。


## SDL实现参考

为什么参考SDL
> 当前Anbox是以及SDL来实现的声音输出。这样能够看到完整的audio处理流程

> SDL有广泛的应用，在声音处理的软件实现质量上肯定是过关的。


### 动态库/静态库加载

主要是加载AlsaSoundLib，因为SDL是一个跨平台的库，所以库的加载都需要SDL自己来做
本次是限定在特定平台下，目前阶段可以暂时不用关注。

### 实现接口一览表


API | 功能 | 实现 | 备注
---|---|---|---
ALSA_OpenDevice | 加载动态链接库，打开设备，并做初始化的配置，和Alsa的Sample是基本一致的|有三类参数，调用者期望的参数配置，用户设定的参数配置，以及alsa设备支持的参数配置，这些需要逐步来匹配设置，策略就是调用者没有设定的，使用环境变量里配置的，最终通过alsa设备的实际支持度来调整。另外一部分主要工作是要调用alsa接口进行初始参数配置|无
ALSA_PlayDevice | 将数据送入alsapcm设备中 | | 无
ALSA_WaitDevice | 等待audio的环形buffer可用空闲长度达到1024 | |无
ALSA_GetDeviceBuf | 返回SDL的隐藏buffer|SDL维护一个buffer，使用这个buffer来向Alsa来写数据，SDL在写数据的过程中，当是6声道的时候，做了数据的处理 | 无
ALSA_CloseDevice| |延时，并调用ALSA_snd_pcm_close完成pcm设备的close | 有buffer相关资源的释放操作
ALSA_Deinitialize |关闭热插拔监测线程，卸载动态链接库 | | 无
ALSA_CaptureFromDevice| | |目前只需要关注palyback即可 
ALSA_FlushCapture| | | 目前只需要关注playback即可
ALSA_DetectDevices|启动线程来检测，使用ALSA_snd_device_name_get_hint来找到设备使用HW，Defaults等默认标志等，找到PCM设备(是否只找PCM设备要确认) | |目前这块应该是不用关注，因为我们可以使用默认设备来做，非SDL定位


---

## Anbox→SDL→Alsa处理流程调查


### SDL:AudioSink设备管理对象(核心数据结构)

SDL内部使用Device来管理AudioDevice，如果多个调用者打开同一个设备，也是允许的，只要alsa能够支持即可。


```
    device->id = id + 1; 初始传入2，SDL内部遍历可用的Audio设备，把ID保存到此处
    device->spec = *obtained; 经过SDL检查计算后的AudioSpec记录到此处，这里用来设定audio的采用率、规格等。同时这里还记录了用于回调的callback函数指针
    device->iscapture = iscapture ? SDL_TRUE : SDL_FALSE；是否打开Capture功能
    device->handle = handle; ：目前没有做什么，目前还是空
    device->shutdown：0 SDL内部的标志位，初始为0
    device->paused：1 SDL内部的标志位，初始为1
    device->enabled：1 SDL内部的标志位，初始为1
    device->mixer_lock；这个用来进行互斥访问soundBuffer使用
    device->callbackspec:这里是重新计算后的AudioSpec。当发生format/channel、samples变更的时候，需要重新计算
    device->work_buffer_len：audiobuffer的长度
    device->work_buffer：存放audio数据
    device->hidden->pcm_handle：这个是隐藏区域，干啥的？？★Todo★
    device->hidden->swizzle_func = swizzle_alsa_channels 这里设置转换函数，估计是BuildStream即流转换的时候才需要，再调查
    device->hidden->mixbuf ：申请MixBuffer，这个是用来干什么的？★Todo★
    device->hidden->mixlen
    device->threadid:SDL_RunAudio函数所在的ThreadID
    device->stream:用于AudioConvert的Stream指针

```

### Anbox->SDL:打开SDL_AudioSink设备

从Anbox调用SDL接口，传递如下的默认配置参数

```
  SDL_memset(&spec_, 0, sizeof(spec_));
  spec_.freq = 44100;
  spec_.format = AUDIO_S16;
  spec_.channels = 2;
  spec_.samples = 1024;
  spec_.callback = &AudioSink::on_data_requested;
  spec_.userdata = this;

  device_id_ = SDL_OpenAudioDevice(nullptr, 0, &spec_, nullptr, 0);
  
  * AUDIO_S16等价于AUDIO_S16LSB，即16位有符号的Little Endian Byte
  * 采用率即44100，标准采用率
  * channel采用双声道
  * samples，这个是什么意思不清楚，目前默认是1024 *Todo*

  
  SDL_OpenAudioDevice(const char *device, int iscapture,
                    const SDL_AudioSpec * desired, SDL_AudioSpec * desired,
                    int allowed_changes)
  
  * 第二个参数默认为0，即使用播放的形式打开，不使用capture
  * 第三个参数传递的是当前使用sdl-audio期望的设备打开参数
  * 第四个参数返回的是一个内部可以支持的sdl-audio的参数，会根据环境变量配置以及Alsa打开时确定的硬件参数确定，因为这里传递了默认的nullptr，所以不关注
  * 第五个参数传递的是0，即不允许变更，也就是音频流不允许在SDL内部做格式转换(如采用率转换)
  
  
```

### Anbox->SDL:暂停SDL_AudioSink设备

```
  device_id_ = SDL_OpenAudioDevice(nullptr, 0, &spec_, nullptr, 0);
  if (!device_id_)
    return false;

  SDL_PauseAudioDevice(device_id_, 0);

```


### SDL:打开SDL_AudioSink设备

> 此处解析会将SDL内部为了管理多个Audio设备的处理逻辑忽略

主要有如下几个步骤

1. 构造SDL_Alsa对象，如果是动态库，则加载动态库
2. SDL内部管理一个DeviceID的数组，找到一个未被专用的Device

```
SDL为多个终端应用来管理多个音频设备，所以做了很多Audio的处理。
本次我们是否也要考虑做一个通用的Audio管理模块？
->经过确认，本次不需要做一个通用的Audio管理模块，主要有如下三个步骤：
1. X86平台上出声
2. 移植到本次目标平台上，出声  ←本次开发目标
3. 设计一个通用的Audio管理模块  ★次期开发准备内容，目前不作为目标

```

3. 决定Alsa设备打开所需的参数校验和准备
```
  prepare_audiospec(desired, obtained)
  * desired:是Anbox传入的参数
  * obtained:是当前系统能够支持的参数
  主要处理逻辑是做下参数检查，如果SDL的调用者未设置，则根据环境变量来获取默认值
  
  其中SDL_CalculateAudioSpec中计算了size，即
    spec->size = SDL_AUDIO_BITSIZE(spec->format) / 8; 16位，计算之后是2
    spec->size *= spec->channels; 计算之后是2*2=4
    spec->size *= spec->samples; 计算之后是4*1024，即4K？
  
  传入一个期望的AudioSpec，根据当前的设备情况判断能够支持的AudioSpec

  目前看到的就是根据输入进行入参判断，如果没有设置，则从环境变量来获取
  如果从环境变量没有获取到数值，则设置为默认值。
  
```

4. 打开AlsaAudio设备

这里是一系列Alsa接口的调用，Open之后是进行参数设定。要结合API说明，详细了解各个函数的意思
在时序图中的编号是从1.1.1.1.1->1.1.1.1.16都是alsa的调用。

```
    impl.OpenDevice(device, handle, devname, iscapture)
    * device是sdl内部维护的audiodevice对象
    * handle根据目前的逻辑，应该是null
    * devname：从环境变量获取的，目前不知道怎么获取
    * iscapture：0，默认使用playback的方式打开
```

这里申请了hidden区域：并将hidden区域纳入device管理
```
struct SDL_PrivateAudioData
{
    /* The audio device handle */
    snd_pcm_t *pcm_handle;

    /* Raw mixing buffer */
    Uint8 *mixbuf;
    int mixlen;

    /* swizzle function */
    void (*swizzle_func)(_THIS, void *buffer, Uint32 bufferlen);
};
```

OpenDevice的内部实现就是完成alsa的api调用，下面详细说明

**step1**:ALSA_snd_pcm_open

> ref

```
Opens a PCM.

Parameters
    pcmp	Returned PCM handle
    name	ASCII identifier of the PCM handle
    stream	Wanted stream
    mode	Open mode (see SND_PCM_NONBLOCK, SND_PCM_ASYNC)
Returns
    0 on success otherwise a negative error code
```

> 调用实例

```
    /* Open the audio device */
    /* Name of device should depend on # channels in spec */
    status = ALSA_snd_pcm_open(&pcm_handle,
                get_audio_device(handle, this->spec.channels),
                iscapture ? SND_PCM_STREAM_CAPTURE : SND_PCM_STREAM_PLAYBACK,
                SND_PCM_NONBLOCK);
    get_audio_device 获取devName，这里还有疑问，devname是怎么获得呢？
    SND_PCM_STREAM_CAPTURE : SND_PCM_STREAM_PLAYBACK：是input还是output，这里选择了SND_PCM_STREAM_PLAYBACK
    SND_PCM_NONBLOCK：以非阻塞的方式打开的
    
```

get_audio_device的内部实现是：根据这个实现逻辑，推测其最终返回的是"default"
也就是alsa提供的设备名称,alsa是使用设备名称来使用的，而不是使用dev/***设备文件的方式
使用设备名称，可以是真实的设备，也可以是虚拟的设备，如plug:**

```
    get_audio_device(void *handle, const int channels)
    {
        const char *device;
    
        if (handle != NULL) {
            return (const char *) handle;
        }
    
        /* !!! FIXME: we also check "SDL_AUDIO_DEVICE_NAME" at the higher level. */
        device = SDL_getenv("AUDIODEV");    /* Is there a standard variable name? */
        if (device != NULL) {
            return device;
        }
    
        if (channels == 6) {
            return "plug:surround51";
        } else if (channels == 4) {
            return "plug:surround40";
        }
    
        return "default";
    }
                
```
> 经过上述处理，将返回的pcm_handle保存到this->hidden->pcm_handle = pcm_handle;


**step2**:获取alsa设备配置ALSA_snd_pcm_hw_params_any

> ref

```
Fill params with a full configuration space for a PCM.

Parameters
    pcm	PCM handle
    params	Configuration space
The configuration space will be filled with all possible ranges for the PCM device.
```

> 调用实例

```
    snd_pcm_hw_params_alloca(&hwparams);
    status = ALSA_snd_pcm_hw_params_any(pcm_handle, hwparams);

```

**step3**:ALSA_snd_pcm_hw_params_set_access

> ref

```
Restrict a configuration space to contain only one access type.

Parameters
    pcm	PCM handle
    params	Configuration space
    access	access type
Returns
    0 otherwise a negative error code if configuration space would become empty
```

> 调用实例

```
/* SDL only uses interleaved sample output */
    status = ALSA_snd_pcm_hw_params_set_access(pcm_handle, hwparams,
                                               SND_PCM_ACCESS_RW_INTERLEAVED);
    if (status < 0) {
        return SDL_SetError("ALSA: Couldn't set interleaved access: %s",
                     ALSA_snd_strerror(status));
    }
```

**step4**:ALSA_snd_pcm_hw_params_set_format

> ref

```
Restrict a configuration space to contain only one format.

Parameters
    pcm	PCM handle
    params	Configuration space
    format	format
Returns
    0 otherwise a negative error code
```

> 调用实例

```
    status = ALSA_snd_pcm_hw_params_set_format(pcm_handle,
                                                       hwparams, format);
    if (status < 0) {
            test_format = SDL_NextAudioFormat();
    }
    此处format：SND_PCM_FORMAT_S16_LE
```


**step5**:ALSA_snd_pcm_hw_params_set_channels

> ref

```
Restrict a configuration space to contain only one channels count.

Parameters
    pcm	PCM handle
    params	Configuration space
    val	channels count
Returns
    0 otherwise a negative error code if configuration space would become empty
    
```

> 调用实例

```
 /* Set the number of channels */
    status = ALSA_snd_pcm_hw_params_set_channels(pcm_handle, hwparams,
                                                 this->spec.channels);
    channels = this->spec.channels;
    if (status < 0) {
        status = ALSA_snd_pcm_hw_params_get_channels(hwparams, &channels);
        if (status < 0) {
            return SDL_SetError("ALSA: Couldn't set audio channels");
        }
        this->spec.channels = channels;
    }
```

**step6**:ALSA_snd_pcm_hw_params_set_rate_near

> ref

```
Restrict a configuration space to have rate nearest to a target.

Parameters
    pcm	PCM handle
    params	Configuration space
    val	approximate target rate / returned approximate set rate
    dir	Sub unit direction
Returns
    0 otherwise a negative error code if configuration space is empty
target/chosen exact value is <,=,> val following dir (-1,0,1)
    
```

> 调用实例

```
    /* Set the audio rate */
    rate = this->spec.freq;
    status = ALSA_snd_pcm_hw_params_set_rate_near(pcm_handle, hwparams,
                                                  &rate, NULL);
    if (status < 0) {
        return SDL_SetError("ALSA: Couldn't set audio frequency: %s",
                            ALSA_snd_strerror(status));
    }
    this->spec.freq = rate;
```

**step7**:设置缓冲buffer


> ref

```
snd_pcm_hw_params_set_period_size_near()

Restrict a configuration space to have period size nearest to a target.

Parameters
    pcm	PCM handle
    params	Configuration space
    val	approximate target period size in frames / returned chosen approximate target period size
    dir	Sub unit direction
Returns
    0 otherwise a negative error code if configuration space is empty
target/chosen exact value is <,=,> val following dir (-1,0,1)
    
snd_pcm_hw_params_set_periods_min    
    
Restrict a configuration space with a minimum periods count.

Parameters
    pcm	PCM handle
    params	Configuration space
    val	approximate minimum periods per buffer (on return filled with actual minimum)
    dir	Sub unit direction (on return filled with actual direction)
Returns
0 otherwise a negative error code if configuration space would become empty    
 
 
 
snd_pcm_hw_params_set_periods_first
Restrict a configuration space to contain only its minimum periods count.

Parameters
    pcm	PCM handle
    params	Configuration space
    val	Returned approximate minimum periods per buffer
    dir	Sub unit direction
Returns
0 otherwise a negative error code

```

> 调用实例

```
   /* Copy the hardware parameters for this setup */
    snd_pcm_hw_params_alloca(&hwparams);
    ALSA_snd_pcm_hw_params_copy(hwparams, params);

    /* Attempt to match the period size to the requested buffer size */
    /* 这里设置了1024个Frames*/
    persize = this->spec.samples;
    status = ALSA_snd_pcm_hw_params_set_period_size_near(
                this->hidden->pcm_handle, hwparams, &persize, NULL);
    if ( status < 0 ) {
        return(-1);
    }

    /* Need to at least double buffer */
    periods = 2;
    status = ALSA_snd_pcm_hw_params_set_periods_min(
                this->hidden->pcm_handle, hwparams, &periods, NULL);
    if ( status < 0 ) {
        return(-1);
    }

    status = ALSA_snd_pcm_hw_params_set_periods_first(
                this->hidden->pcm_handle, hwparams, &periods, NULL);
    if ( status < 0 ) {
        return(-1);
    }

    /* "set" the hardware with the desired parameters */
    status = ALSA_snd_pcm_hw_params(this->hidden->pcm_handle, hwparams);
    if ( status < 0 ) {
        return(-1);
    }
```

**step8**:设置软件参数(驱动参数)

> ref

```
snd_pcm_sw_params_set_avail_min

Set avail min inside a software configuration container.

Parameters
    pcm	PCM handle
    params	Software configuration container
    val	Minimum avail frames to consider PCM ready
Returns
    0 otherwise a negative error code
Note: This is similar to setting an OSS wakeup point. The valid values for 'val' are determined by the specific hardware. Most PC sound cards can only accept power of 2 frame counts (i.e. 512, 1024, 2048). You cannot use this as a high resolution timer - it is limited to how often the sound card hardware raises an interrupt.
```

```
snd_pcm_sw_params_set_start_threshold

Set start threshold inside a software configuration container.

Parameters
    pcm	PCM handle
    params	Software configuration container
    val	Start threshold in frames
Returns
    0 otherwise a negative error code
PCM is automatically started when playback frames available to PCM are >= threshold or when requested capture frames are >= threshold

```


> 调用实例

```
   /* Set the software parameters */
    snd_pcm_sw_params_alloca(&swparams);
    status = ALSA_snd_pcm_sw_params_current(pcm_handle, swparams);
    if (status < 0) {
        return SDL_SetError("ALSA: Couldn't get software config: %s",
                            ALSA_snd_strerror(status));
    }
    status = ALSA_snd_pcm_sw_params_set_avail_min(pcm_handle, swparams, this->spec.samples);
    if (status < 0) {
        return SDL_SetError("Couldn't set minimum available samples: %s",
                            ALSA_snd_strerror(status));
    }
    status =
        ALSA_snd_pcm_sw_params_set_start_threshold(pcm_handle, swparams, 1);
    if (status < 0) {
        return SDL_SetError("ALSA: Couldn't set start threshold: %s",
                            ALSA_snd_strerror(status));
    }
    status = ALSA_snd_pcm_sw_params(pcm_handle, swparams);
    if (status < 0) {
        return SDL_SetError("Couldn't set software audio parameters: %s",
                            ALSA_snd_strerror(status));
    }
```



### 构建转换Stream

转换前提条件：当如下的参数不匹配，即上层不支持的时候，则需要做转换
* device->spec.freq
* device->spec.format
* device->spec.samples 
* device->spec.channels

Audio数据转换的处理逻辑比较底层和复杂，所以需要判断是否需要再决定是否继续调查

> 20191015追记
从现有的SDL接口设计来看，目前可以作为灵活的接口供使用者来使用。所以此处不影响基本流。
则优先度可以降低。



### SDL_RunAudio

主要处理逻辑
1. SDL_SetThreadPriority(SDL_THREAD_PRIORITY_TIME_CRITICAL)
2. ThreadInit
3. 进入主循环

```
概要处理逻辑是
0. 将数据从app通过callback读入mixbuffer
1. 从mixbuffer中读取数据做转换
2. 送入AudioSound内的Ringbuffer
3. 等待进入下一轮
```

这里分两种情况：
1. AudioStream不需要转换
```
  1. 获取Mixbuffer的首地址
  2. 调用Callback来拷贝数据到mixbuffer
  3. PlayDevice
  4. WaitDevice
```

2. AudioStream需要转换

```
  1. 获取WorkBuffer的首地址
  2. 调用Callback来拷贝数据到WorkBuffer
  3. 从WorkBuffer获取数据送入AudioStream转换Stream
  3. 等待转换完成SDL_AudioStreamAvailable
  4. 分支
        4.1 转换失败，什么都不做
        4.2 转换成功，将数据从AudioStream转换stream取出送到mixBuffer
        4.3 PlayDevice→WaitDevice
  
```


#### PlayDevice

这里调用了Alsa，实现了数据传送到AudioDriver，并等待的处理
因为这里需要涉及到Audio格式引起的步长的计算，所以此处将SDL的代码贴上，并添加说明

```
ALSA_PlayDevice(_THIS)
{
    const Uint8 *sample_buf = (const Uint8 *) this->hidden->mixbuf;
    * 这是获取mixBuffer的起始地址
    const int frame_size = (((int) SDL_AUDIO_BITSIZE(this->spec.format)) / 8) *
                                this->spec.channels;
    * 这里当前是有符号16位LSB，所以这里计算后frameSize是2*2(立体声2声道)，即=4
    snd_pcm_uframes_t frames_left = ((snd_pcm_uframes_t) this->spec.samples);
    * frames_left这里被设置为1024个frame
    
    this->hidden->swizzle_func(this, this->hidden->mixbuf, frames_left);
    * 此处做数据的转换，转换是简单的逻辑，主要是LSB/MSB等的转换，或者是左声道/右声道的数据组织方式差异的转化
    * 因为当前声道采用2声道，所以此处没有做任何处理

    while ( frames_left > 0 && SDL_AtomicGet(&this->enabled) ) {
        * 这里向PCM设备写入数据，每次尝试写入全部数据，如果部分写入的情况发生，则再次尝试写入
        int status = ALSA_snd_pcm_writei(this->hidden->pcm_handle,
                                         sample_buf, frames_left);

        if (status < 0) {
            * 这里是超过了Alsa设备中的环形buffer的容量
            if (status == -EAGAIN) {
                /* Apparently snd_pcm_recover() doesn't handle this case -
                   does it assume snd_pcm_wait() above? */
                SDL_Delay(1);
                continue;
            }
            * 如果不是超出了环形buffer，则可能是pcm设备出错，需要尝试恢复
            status = ALSA_snd_pcm_recover(this->hidden->pcm_handle, status, 0);
            if (status < 0) {
                /* Hmm, not much we can do - abort */
                fprintf(stderr, "ALSA write failed (unrecoverable): %s\n",
                        ALSA_snd_strerror(status));
                SDL_OpenedAudioDeviceDisconnected(this);
                return;
            }
            continue;
        }
        else if (status == 0) {
            /* No frames were written (no available space in pcm device).
               Allow other threads to catch up. */
            Uint32 delay = (frames_left / 2 * 1000) / this->spec.freq;
            SDL_Delay(delay);
        }
        * mixbuffer里面放置了准备好的audio数据，按照audio增长步长来写入数据
        sample_buf += status * frame_size;
        frames_left -= status;
    }
}
```

> ref

```
Write interleaved frames to a PCM.

Parameters
    pcm	PCM handle
    buffer	frames containing buffer
    size	frames to be written
Returns
    a positive number of frames actually written otherwise a negative error code
Return values
    -EBADFD	PCM is not in the right state (SND_PCM_STATE_PREPARED or SND_PCM_STATE_RUNNING)
    -EPIPE	an underrun occurred
    -ESTRPIPE	a suspend event occurred (stream is suspended and waiting for an application recovery)

If the blocking behaviour is selected and it is running, then routine waits until all requested frames are played or put to the playback ring buffer. The returned number of frames can be less only if a signal or underrun occurred.

If the non-blocking behaviour is selected, then routine doesn't wait at all.

The function is thread-safe when built with the proper option.

```

#### ALSA_WaitDevice

这里主要是去Alsa设备处查询可用的环形buffer的长度，当得到1024个frame的时候，就返回。

```
ALSA_WaitDevice(_THIS)
{

    * 希望能够获取到1024个frame
    const snd_pcm_sframes_t needed = (snd_pcm_sframes_t) this->spec.samples;
    
    while (SDL_AtomicGet(&this->enabled)) {
        * 去pcm设备查询，当前可用的环形buffer可用的frame是多少
        const snd_pcm_sframes_t rc = ALSA_snd_pcm_avail(this->hidden->pcm_handle);
        if ((rc < 0) && (rc != -EAGAIN)) {
            /* Hmm, not much we can do - abort */
            fprintf(stderr, "ALSA snd_pcm_avail failed (unrecoverable): %s\n",
                        ALSA_snd_strerror(rc));
            SDL_OpenedAudioDeviceDisconnected(this);
            return;
        } else if (rc < needed) {
           * 计算需要等待多少秒，等待的时间是通过audio的freq来计算的。
            const Uint32 delay = ((needed - (SDL_max(rc, 0))) * 1000) / this->spec.freq;
            SDL_Delay(SDL_max(delay, 10));
        } else {
            break;  /* ready to go! */
        }
    }

}

```

> ref

```
snd_pcm_avail

Return number of frames ready to be read (capture) / written (playback)

Parameters
    pcm	PCM handle
Returns
    a positive number of frames ready otherwise a negative error code

On capture does all the actions needed to transport to application level all the ready frames across underlying layers.

The position is synced with hardware (driver) position in the sound ring buffer in this functions.

The function is thread-safe when built with the proper option.
```


---

经过上述调查，能够形成一个静态的处理流程视图，现在调查缓冲部分，以便形成一个动态的运行视图

---


### Android→SDL→AudioDriver之间的数据缓冲机制

> Android应用(含AndroidAudioLib)会将数据按照不确定的频度来将raw pcm数据写入socket

↓
> BoostAsio使用异步非阻塞的方式，响应Android侧的Socket写入，并将数据缓存到Host端的环形buffer中
这个环形buffer有16个buffer元组构成，每个buffer元组长度是512个字节
使用count来作为游标，使用pos来标记往16个buffer中哪个buffer元组写入

↓
> SDL_RunAudio每次读取Samples*channels*spec.fmt/8,即如果是16位的2channel的，samples设定为1024
则每次会读取1024*4的raw pcm数据
调用Alsa-Driver的writei接口来向AudioDriver来写入数据
查询AudioDriver是否已经有空闲的buffer用来写入数据

↓
> AudioDriver同样维护着一个环形Buffer，要在初始化时设定环形buffer的长度
当audioDriver准备完成后，即读入数据(这里如果不能读入数据，则会一直等待)
另外AudioPCM模块按照指定的速度来读取环形buffer的数据，输出声音。


问题：如何打开，如何关闭，如何重启呢？

> 调查结果：这里设备初始打开后，就一直打开，不再关闭也不需要重启，除非使用者调用了close函数。
其实现是sdl回调callback函数时，如果没有数据，会一直等待，并不需要调用者给与太多控制了。
也就是说，SDL傻瓜式的认为一直有数据，就一直往Audio设备送。
当没有数据，也就是回调函数不再返回，这个sdl不再关注了。只要回调函数返回，即送入audio数据。


> 从上面的实现路径来看，因为采用BoostAsio的异步非阻塞特性，要求在socket读的异步回调中不能出现耗时的操作
同时Android侧输出Audio数据是不连续的，而送入AudioDriver的数据需要是是均匀的数据段，所以这里需要有一定的缓冲


#### 运行模型检讨

> 如果不限定送入AudioDriver的数据长度，也不启动线程，即采用纯Lib库形式而不引入新的线程是不是可以呢？
想定的实现逻辑如下所示：
1. 响应BoostAsio的socket写入通知，触发socket的读取数据回调
2. 在回调中沿用现有的处理逻辑，直接向数据存入环形buffer
3. 从环形buffer中读取指定数据的长度(三种情况：1）音频刚开始播放，此时要等待足够多的数据；2）当音频播放中，仍然等待足够多的数据；3）当音频播放到末尾，应该立即返回，此时要怎么来区分呢？)
4. 判断是否需要向AudioDriver写入数据(当前环形buffer有sample标识长度的空闲位置)
5. Step4中如果可以写入，则将从环形buffer读的数据，送入audioDriver的环形buffer中。
这里的问题就是如何来判断raw pcm数据已经到了末尾，直接将buffer中的数据快速送入audioDriver中呢？
→参考中实现的逻辑是当已经读到数据，而且环形buffer中已经没有数据了，此时就直接送入audioDriver，从这个逻辑来看，好像没问题。
这样看上去，也能实现，那么唯一的问题就是在BoostAsio中的异步回调函数中，是否支持这么多的操作处理，会否会影响音频的连续播放，问题看上去比较复杂，如果某个回调函数的执行时间比较长，则可能会引起在多个回调函数的互斥访问。

> 这样从现实的角度，还是采用异步的编程模型更合理，需要新启动一个线程来完成送入数据到AudioBuffer的操作
采用这样的编程模型，比较稳妥。不好的地方是需要加上多个同步互斥操作。比较麻烦。

---

> 设计要点：
ADL(Audio Driver Layer)需要提供异步编程模型，以及同步编程模型，以便后续调整，目前阶段完成了异步模型。

---



### Audio Spec传递流程

Audio PCM数据的输出，有两个要点：
1. PCM数据的AudioSpec是清楚的准确的，即对AudioDrvier来说，输入数据的Spec和设定的Spec（AudioDriver也是支持的Spec）完全一致，则声音即可出来
2. 在Timing方面控制要精确，避免出现杂音以及中断



> Android侧在调用write函数时，hal层映射的是out_write函数
在这个函数中通过stuct audio_stream_out * stream来传入当前的音频的参数

↓

> Anbox使用默认的格式来设定sdl，这里潜在的问题是如果更换为其他的音频来源，则可能当前的实现就是不支持的

↓

> SDL使用Anbox-host侧默认的audio参数来设定audioSpec，SDL根据这个audioSpec来进行alsa Audio的设定，当audioSpec不支持的时候，SDL内部启用audio Covert来处理




### SDL_Hotplug Dectect

Audio设备的热插拔


### 多设备管理

当前SDL只要检查名字对的上，就可以再行打开
当然是否能够打开，需要Alsa支持。

目前Alsa支持的情况是：依赖打开设备名称，如果设备名称是一个软的plughw，则可以重复打开。如果打开的是一个直接的硬件，则不能以共享方式的打开。

SDL使用一个数组来维护当前打开的设备，当调用者调用open_Audiodevice打开设备时，如果能够打开，则所有的如RunAudio这个主循环都是一整套的都建立起来，如有两个调用者同时打开同一个设备，则SDL内部是有两个主线程来pull数据，并将数据送至alsa中。


### 多个音源Mix处理

mix的问题是使用sdl开放出来的接口来做的audio mix，是在使用者层面来做的，没有在sdl_audio的主流程来做。这样就简单了，当前可以不关注


### 接口的设计

参考：SDL的接口目前都在sdl_Audio.h中，在这个文件中，提供了如下几类接口：1. 资源申请以及playback、capture；2.audioconvert；3.audioStream；4.load wav file接口；5. playback用的同步接口

其中第2类和第3类是之前没有理解到的，这个可以在使用者层面来调用，这样可以将使用者层面来确保audio Spec的in和out一致。然后简单的使用alsa的接口简单的将raw pcm数据输出到alsa设备中，完成pcm播放


SDL有两种接口：异步回调风格的接口，以及同步接口
PulseAudio：目前看到的是使用异步回调接口
当前车载系统Linux一般使用的是pulseaudio来实现的，pulseaudio实现了audio的spec转换、mix功能，音量调整功能。再通过audioDSP来实现音源切换、以及音效功能

从这样看，模拟SDL的接口问题不大，不会出问题





---
设计过程
---

## 现状

Anbox->Host>SDL->Alsa能够音声出力。
如果是奋斗的小鸟，则可能不会出来，这里涉及到audio的spec差异问题。
2019/10/5追记：

> 安卓内部加载audiohw的动态库，查询当前驱动的处理能力，然后进行相关audio的解码调制工作
输入给audiohw的可以是符合目标需求的pcm流。如此就不会出现奋斗的小鸟以差异化的audiospec来播放了。


## 目标

目标1：能够完成Anbox在EVB上使用Alsa立上
> X86：固定的AudioSpec、固定的DevName，简单的控制流，不包含audio convert、audio mix
※如果需要依赖Audio Convert，则AudioConvert是可以软插拔的，不是实现关键路径上的节点

目标2：能够完成Anbox上多种声音的输出

> X86：可变的AudioSpec，固定的DevName，简单的控制流
不包含Audio Convert以及AudioMix
* 如果需要使用AudioConvert，则要求AudioConvert是软插拔支持的

> 2019//11/5追记
根据当前的调查结果，安卓系统内可以做audio的解码和调制，所以完成了目标1后，目标2也就可以实现了。


目标3可选：能够配合完成整体的Anbox改造

> X86: 提供同步接口，实现Reactor模型的同步非阻塞模型的通信，所以需要提供同步调用接口


目标4可选：提供一个完整的Audio管理模块

> 需求要包括AudioCapture、Audio Playback、AudioMixer、AudioEffect、AudioVolume、Audio Input Select
※目前这里只涉及到playback，capture，volume以及mixer，其他的都是audio dsp的要实现的功能

## 需求

需求：audio主要关联几个功能：
 采集、输出、音量调整、Mix、音效、切换声音源
* 采集：alsa，基于alsa来操作pcm数据
* 数据：alsa，基于alsa来操作pcm数据
* 音量调整：软件实现，也可以通过plugin来实现
* Mix：一般是软件来实现，也可以通过plugin来实现
* 音效：一般是附带的dsp来实现
* 切换音源：一般是dsp来实现
* Audiospec转换：一般是软件实现，SDL也有类似的实现功能


## 技术方案

参考SDL实现是比较合理的方式，实现方针是参考SDL完成相关的Audio播放功能。


## API Design

参考“附录：API Design Reference”，做API Design


## 实现步骤

1. init、Open、Run、Pause 【完】
2. 实现DVB上的Alsa驱动组入:【完】
3. close：OK，在Close中调用join等待work线程结束。【完】
4. 在X86上调试
5. 在EVB上调试

---
以下是目标3和目标4所依赖的目标不作为高优先级实现
6. audioConvert 函数形式提供 【目前看不需要做audio转换，所以可以】
7. 同步接口
8. multiDevice管理
9. audioMixer 函数形式提供


## 实现基本要点

1. 数据类型参考stdint.h来定义来修改；
2. 基础设施使用C++11语言层面提供的基础设施来实现，比如atomic、thread、mutex
3. 日志使用anbox中的日志系统：目前还有残留工作，日志上有若干不符合release标准的日志存在
4. 构成，sdl_audio.c 和sdlalsa形成两个类，还有sdlaudiospec需要新构造类，其他待补充
5. 如果将sdl_audio形成一个类，则这个类将会是一个非常大的类。后续根据编码过程中的理解再看下是否可以拆解：现状OK。(目前看不太大，因为没有包含covertmix以及同步等功能，后续增加的时候考虑如何来做。)
6. 同步以及互斥的操作，都尽量保持.使用boost库的实现方式来替换
7. 现有结构要保持对audiocvt的兼容，后续可能会追加。
8. 考虑两种接口，一个是当前的异步回调接口(sdl的实现方式)，还有一个是同步接口，面向将来使用同期非阻塞模型是来使用:下一步实施



## 附录：API Design Reference

### SDL_ConvertAudio

SDL_ConvertAudio takes one parameter, cvt, which was previously initilized. Initilizing a SDL_AudioCVT is a two step process. First of all, the structure must be passed to SDL_BuildAudioCVT along with source and destination format parameters. Secondly, the cvt->buf and cvt->len fields must be setup. cvt->buf should point to the audio data and cvt->len should be set to the length of the audio data in bytes. Remember, the length of the buffer pointed to by buf show be len*len_mult bytes in length.

Once the SDL_AudioCVTstructure is initilized then we can pass it to SDL_ConvertAudio, which will convert the audio data pointer to by cvt->buf. If SDL_ConvertAudio returned 0 then the conversion was completed successfully, otherwise -1 is returned.

If the conversion completed successfully then the converted audio data can be read from cvt->buf. The amount of valid, converted, audio data in the buffer is equal to cvt->len*cvt->len_ratio.

> SDL API design Reference

```
/* Converting some WAV data to hardware format */
void my_audio_callback(void *userdata, Uint8 *stream, int len);

SDL_AudioSpec *desired, *obtained;
SDL_AudioSpec wav_spec;
SDL_AudioCVT  wav_cvt;
Uint32 wav_len;
Uint8 *wav_buf;
int ret;

/* Allocated audio specs */
desired = malloc(sizeof(SDL_AudioSpec));
obtained = malloc(sizeof(SDL_AudioSpec));

/* Set desired format */
desired->freq=22050;
desired->format=AUDIO_S16LSB;
desired->samples=8192;
desired->callback=my_audio_callback;
desired->userdata=NULL;

/* Open the audio device */
if ( SDL_OpenAudio(desired, obtained) < 0 ){
  fprintf(stderr, "Couldn't open audio: %s\n", SDL_GetError());
  exit(-1);
}
        
free(desired);

/* Load the test.wav */
if( SDL_LoadWAV("test.wav", &wav_spec, &wav_buf, &wav_len) == NULL ){
  fprintf(stderr, "Could not open test.wav: %s\n", SDL_GetError());
  SDL_CloseAudio();
  free(obtained);
  exit(-1);
}
                                            
/* Build AudioCVT */
ret = SDL_BuildAudioCVT(&wav_cvt,
                        wav_spec.format, wav_spec.channels, wav_spec.freq,
                        obtained->format, obtained->channels, obtained->freq);

/* Check that the convert was built */
if(ret==-1){
  fprintf(stderr, "Couldn't build converter!\n");
  SDL_CloseAudio();
  free(obtained);
  SDL_FreeWAV(wav_buf);
}

/* Setup for conversion */
wav_cvt.buf = malloc(wav_len * wav_cvt.len_mult);
wav_cvt.len = wav_len;
memcpy(wav_cvt.buf, wav_buf, wav_len);

/* We can delete to original WAV data now */
SDL_FreeWAV(wav_buf);

/* And now we're ready to convert */
SDL_ConvertAudio(&wav_cvt);

/* do whatever */
.
.
.
.


```

### Using SDL_AudioStream

From the dawn of time, until SDL 2.0.6, there was only one way to convert audio through SDL: By using the SDL_AudioCVT structure.

It's a usable API, for various needs, but it has a few problems:

    It's hard to understand how to use.
    It can't carry any dynamic state; there's no API to "free" a structure, so it can't allocate memory, or do special things for various converters. In 2.0.6, it even commandeers a pointer in the struct it hopes you aren't using, just to give it space for a few more bits of internal state information. This also leads to some other inefficient tapdances to wedge functionality into it.
    The existing structure is extremely rigid, expecting certain fields to be set by the app and certain fields to be set by SDL, and there is no room for expansion. Did I mention we steal a pointer field?
    Perhaps most importantly: it can't resample data in chunks. You have to give it all the data it expects in one shot, or you'll get gaps and skips in your audio output. 

We have a better API that SDL has been using internally for awhile now, since it needs to bridge data between the app's audio callbacks and the platform APIs that consume and produce data, and that data might be coming and going at any size and format at inexact times. Not only does this API have to convert and resample data on the fly, it needs to be able to buffer it when one end produces data at a different rate than the other is consuming it.

For SDL 2.0.7, we've cleaned up these internal APIs and made them available to apps. We call it SDL_AudioStream.

To avoid confusion: this is strictly an optional API, even if you use SDL for audio playback or capture. SDL might use it behind the scenes if it silently converts data between your callback and the platform, but that isn't your concern. If you don't like callbacks and just wanted to feed SDL audio data as you have more to give it, and let SDL figure it out, you can do that too, but that's a different API (that's SDL_QueueAudio() and friends).

Here are some immediate uses for SDL_AudioStream:

    You want to decode an Ogg Vorbis file, and convert it to a specific format for playback on the fly, but libvorbis only hands you 512 bytes of uncompressed data at a time. You can push it through an SDL_AudioStream a drip at a time, and pull converted data out of the other side, in a different format, as needed.
    You have a VoIP app, but you can't promise when audio packets will arrive over the network, or how large those packets will be when they do, or if you'll have to replace them with a chunk of silence when they don't arrive at all...but you want to be able to produce a single coherent stream of audio data, ready for playback.
    You want to pull audio data from disk as fast as possible, stuff it into something that will do the proper conversions, and dump the results back to disk without having to worry about the details of the data as it's passing through.
    You want to write a mixer that only deals with a single specific format and have it output to whatever else might need to eat that data.
    You have some procedurally-generated sound and want to produce more when most of it has been consumed.
    You have no interest in converting audio to a different format, but you just want someone else to worry about storing it in an efficient, queue-like data structure until you are ready to make use of it. 

Using SDL_AudioStream is pretty simple. First, you create one. Let's say you want to produce mono data in Sint16 format at 22050Hz, for something that wants to consume stereo data in Float32 format at 48000Hz.

```
// You put data at Sint16/mono/22050Hz, you get back data at Float32/stereo/48000Hz
SDL_AudioStream *stream = SDL_NewAudioStream(AUDIO_S16, 1, 22050, AUDIO_F32, 2, 48000);
if (stream == NULL) {
    printf("Uhoh, stream failed to create: %s\n", SDL_GetError());
} else {
    // We are ready to use the stream!
}

```
Now all you have to do is feed your stream data!

```
Sint16 samples[1024];
int num_samples = read_more_samples_from_disk(samples); // whatever.
// you tell it the number of _bytes_, not samples, you're putting!
int rc = SDL_AudioStreamPut(stream, samples, num_samples * sizeof (Sint16));
if (rc == -1) {
    printf("Uhoh, failed to put samples in stream: %s\n", SDL_GetError());
    return;
}

// Whoops, forgot to add a single sample at the end...!
//  You can put any amount at once, SDL will buffer
//  appropriately, growing the buffer if necessary.
Sint16 onesample = 22;
SDL_AudioStreamPut(stream, &onesample, sizeof (Sint16));
```
As you add data to the stream, SDL will convert and resample it. You can ask how much converted data is available:

```
int avail = SDL_AudioStreamAvailable(stream);  // this is in bytes, not samples!
if (avail < 100) {
    printf("I'm still waiting on %d bytes of data!\n", 100 - avail);
}
```
And when you have enough data to be useful, you can read out samples in the requested format:

```
float converted[100];
// this is in bytes, not samples!
int gotten = SDL_AudioStreamGet(stream, converted, sizeof (converted));
if (gotten == -1) {
    printf("Uhoh, failed to get converted data: %s\n", SDL_GetError());
}
write_more_samples_to_disk(converted, gotten); /* whatever. */
```
Of course, you don't have to read it all at once. This both streams in and out of a converted buffer, so you can read less than is available:

```
int gotten;
do {
    float converted[100];
    // this is in bytes, not samples!
    gotten = SDL_AudioStreamGet(stream, converted, sizeof (converted));
    if (gotten == -1) {
        printf("Uhoh, failed to get converted data: %s\n", SDL_GetError());
    } else {
        // (gotten) might be less than requested in SDL_AudioStreamGet!
        write_more_samples_to_disk(converted, gotten); /* whatever. */
    }
} while (gotten > 0);
```
In terms of performance: buffer allocations, conversion, and resampling happen during stream puts. Getting from the stream is a little bookkeeping and some memcpy() calls. Plan accordingly.

The one gotcha of this interface: you might notice that you have less available than you expect (possibly even zero bytes available!). When resampling, SDL keeps a buffer of padding available so that data sent through in chunks still resamples smoothly. Rather than try to predict the future, it just holds onto the first little piece you feed into the stream, and then starts converting that part after it's received more data, holding a tiny bit back each time to keep the stream sounding smooth.

There are two ways to deal with this: if you're planning to stream forever, don't do anything. Just keep feeding more data as you have it, and reading more data from the stream as it becomes available, and it'll all work out.

If you are simply at the end of the data you want to stream, you can communicate this to SDL and it will convert any buffered data it's been holding onto internally, making it available to be read with SDL_AudioStreamGet().

```
    SDL_AudioStreamFlush(stream);
```
Note that if you flush a stream, you can then feed it more data, but there will likely be gaps in the audio output, as the resampler will use silence for the padding at the end. You really only want to flush to finish off a stream and get the last few samples out of it.

If, for whatever reason, you want to throw a stream's contents away without reading it, you can:

```
    SDL_AudioStreamClear(stream);
```
This will remove any data you've put to the stream without reading, and reset internal state (so the resampler will be expecting a fresh buffer instead of resampling against data you previously wrote to the stream). This is useful if you plan to reuse a stream for different source, or just decided that the current source wasn't working out; maybe you're muting an offensive person on a VoIP app.

When you are done with the stream, you can destroy it:

```

     SDL_FreeAudioStream(stream);
```
This frees up internal state and buffers. You don't have to drain the stream before freeing it. The SDL_AudioStream pointer you've been using is invalid after this call. 

### Opening the audio device

```
    SDL_AudioSpec wanted;
    extern void fill_audio(void *udata, Uint8 *stream, int len);

    /* Set the audio format */
    wanted.freq = 22050;
    wanted.format = AUDIO_S16SYS;
    wanted.channels = 2;    /* 1 = mono, 2 = stereo */
    wanted.samples = 1024;  /* Good low-latency value for callback */
    wanted.callback = fill_audio;
    wanted.userdata = NULL;

    /* Open the audio device, forcing the desired format */
    if ( SDL_OpenAudio(&wanted, NULL) < 0 ) {
        fprintf(stderr, "Couldn't open audio: %s\n", SDL_GetError());
        return(-1);
    }
    return(0);
```

### Play the audio 

```
   static Uint8 *audio_chunk;
    static Uint32 audio_len;
    static Uint8 *audio_pos;

    /* The audio function callback takes the following parameters:
       stream:  A pointer to the audio buffer to be filled
       len:     The length (in bytes) of the audio buffer
    */
    void fill_audio(void *udata, Uint8 *stream, int len)
    {
        /* Only play if we have data left */
        if ( audio_len == 0 )
            return;

        /* Mix as much data as possible */
        len = ( len > audio_len ? audio_len : len );
        SDL_MixAudio(stream, audio_pos, len, SDL_MIX_MAXVOLUME);
        audio_pos += len;
        audio_len -= len;
    }

    /* Load the audio data ... */

    ;;;;;

    audio_pos = audio_chunk;

    /* Let the callback function play the audio chunk */
    SDL_PauseAudio(0);

    /* Do some processing */

    ;;;;;

    /* Wait for sound to complete */
    while ( audio_len > 0 ) {
        SDL_Delay(100);         /* Sleep 1/10 second */
    }
    SDL_CloseAudio();
```

### Playing streaming audio, mixing 2 (or more) audio streams


```
#include <iostream>
#include <cmath>
#include "SDL/SDL.h"
#include "SDL/SDL_main.h"

/* linker options: -lmingw32 -lSDLmain -lSDL -mwindows */

using namespace std;

unsigned int sampleFrequency = 0;
unsigned int audioBufferSize = 0;
unsigned int outputAudioBufferSize = 0;

unsigned int freq1 = 1000;
unsigned int fase1 = 0;
unsigned int freq2 = 5000;
unsigned int fase2 = 0;

void example_mixaudio(void *unused, Uint8 *stream, int len) {

    unsigned int bytesPerPeriod1 = sampleFrequency / freq1;
    unsigned int bytesPerPeriod2 = sampleFrequency / freq2;

    for (int i=0;i<len;i++) {
        int channel1 = int(150*sin(fase1*6.28/bytesPerPeriod1));
        int channel2 = int(150*sin(fase2*6.28/bytesPerPeriod2));

        int outputValue = channel1 + channel2;           // just add the channels
        if (outputValue > 127) outputValue = 127;        // and clip the result
        if (outputValue < -128) outputValue = -128;      // this seems a crude method, but works very well

        stream[i] = outputValue;

        fase1++;
        fase1 %= bytesPerPeriod1;
        fase2++;
        fase2 %= bytesPerPeriod2;
    }
}

int main(int argc, char *argv[])
{

    if( SDL_Init(SDL_INIT_TIMER | SDL_INIT_AUDIO ) <0 ) {
        cout << "Unable to init SDL: " << SDL_GetError() << endl;
        return 1;
    }

    /* setup audio */
    SDL_AudioSpec *desired, *obtained;

    /* Allocate a desired SDL_AudioSpec */
    desired = (SDL_AudioSpec *) malloc(sizeof(SDL_AudioSpec));

    /* Allocate space for the obtained SDL_AudioSpec */
    obtained = (SDL_AudioSpec *) malloc(sizeof(SDL_AudioSpec));

    /* choose a samplerate and audio-format */
    desired->freq = 44100;
    desired->format = AUDIO_S8;

    /* Large audio buffers reduces risk of dropouts but increases response time.
     *
     * You should always check if you actually GOT the audiobuffer size you wanted,
     * note that not hardware supports all buffer sizes (< 2048 bytes gives problems with some
     * hardware). Older versions of SDL had a bug that caused many configuration to use a
     * buffersize of 11025 bytes, if your sdl.dll is approx. 1 Mb in stead of 220 Kb, download
     * v1.2.8 of SDL or better...)
     */
    desired->samples = 4096;

    /* Our callback function */
    desired->callback=example_mixaudio;
    desired->userdata=NULL;

    desired->channels = 1;

    /* Open the audio device and start playing sound! */
    if ( SDL_OpenAudio(desired, obtained) < 0 ) {
        fprintf(stderr, "AudioMixer, Unable to open audio: %s\n", SDL_GetError());
        exit(1);
    }

    audioBufferSize = obtained->samples;
    sampleFrequency = obtained->freq;

    /* if the format is 16 bit, two bytes are written for every sample */
    if (obtained->format==AUDIO_U16 || obtained->format==AUDIO_S16) {
        outputAudioBufferSize = 2*audioBufferSize;
    } else {
        outputAudioBufferSize = audioBufferSize;
    }

    SDL_Surface *screen = SDL_SetVideoMode(200,200, 16, SDL_SWSURFACE);
    SDL_WM_SetCaption("Audio Example",0);

    SDL_PauseAudio(0);

    bool running = true;

    SDL_Event event;
    while (running) {
        while (SDL_PollEvent(&event)) {
            /* GLOBAL KEYS / EVENTS */
            switch (event.type) {
            case SDL_KEYDOWN:
                switch (event.key.keysym.sym) {
                case SDLK_ESCAPE:
                    running = false;
                    break;
                default: break;
                }
                break;
            case SDL_QUIT:
                running = false;
                break;
            }
            SDL_Delay(1);
        }
        SDL_Delay(1);
    }
    SDL_Quit();
    return EXIT_SUCCESS;
}
```

### Play A Wav File

```
#include <SDL2/SDL.h>

#define MUS_PATH "Roland-GR-1-Trumpet-C5.wav"

// prototype for our audio callback
// see the implementation for more information
void my_audio_callback(void *userdata, Uint8 *stream, int len);

// variable declarations
static Uint8 *audio_pos; // global pointer to the audio buffer to be played
static Uint32 audio_len; // remaining length of the sample we have to play


/*
** PLAYING A SOUND IS MUCH MORE COMPLICATED THAN IT SHOULD BE
*/
int main(int argc, char* argv[]){

	// Initialize SDL.
	if (SDL_Init(SDL_INIT_AUDIO) < 0)
			return 1;

	// local variables
	static Uint32 wav_length; // length of our sample
	static Uint8 *wav_buffer; // buffer containing our audio file
	static SDL_AudioSpec wav_spec; // the specs of our piece of music
	
	
	/* Load the WAV */
	// the specs, length and buffer of our wav are filled
	if( SDL_LoadWAV(MUS_PATH, &wav_spec, &wav_buffer, &wav_length) == NULL ){
	  return 1;
	}
	// set the callback function
	wav_spec.callback = my_audio_callback;
	wav_spec.userdata = NULL;
	// set our global static variables
	audio_pos = wav_buffer; // copy sound buffer
	audio_len = wav_length; // copy file length
	
	/* Open the audio device */
	if ( SDL_OpenAudio(&wav_spec, NULL) < 0 ){
	  fprintf(stderr, "Couldn't open audio: %s\n", SDL_GetError());
	  exit(-1);
	}
	
	/* Start playing */
	SDL_PauseAudio(0);

	// wait until we're don't playing
	while ( audio_len > 0 ) {
		SDL_Delay(100); 
	}
	
	// shut everything down
	SDL_CloseAudio();
	SDL_FreeWAV(wav_buffer);

}

// audio callback function
// here you have to copy the data of your audio buffer into the
// requesting audio buffer (stream)
// you should only copy as much as the requested length (len)
void my_audio_callback(void *userdata, Uint8 *stream, int len) {
	
	if (audio_len ==0)
		return;
	
	len = ( len > audio_len ? audio_len : len );
	//SDL_memcpy (stream, audio_pos, len); 					// simply copy from one buffer into the other
	SDL_MixAudio(stream, audio_pos, len, SDL_MIX_MAXVOLUME);// mix from one buffer into another
	
	audio_pos += len;
	audio_len -= len;
}

```






