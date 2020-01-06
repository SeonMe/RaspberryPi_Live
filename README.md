## Raspberry Pi（树莓派）推流直播指南

### 概要：

本着不吃灰的原因，把闲置的树莓派利用起来，做一个 7x24h 的直播姬，原理很简单，使用 FFmpeg 录屏录音进行推流，但是碍于这白给的性能，使用 H.264 进行实时编码推流 Pi 的温度高达 55 度（楼主给自家的 Pi 装了乌龟壳和俩风扇），所以散热很重要，散热很重要，散热很重要。

以及首先需要说明的是，楼主日常使用 Spotify 来听歌，所以想通过在 Pi 上收听 Spotify 后，便滋生了此想法，如果阁下不是使用 Spotify 还望自行想办法。（据说网易云音乐也可以在 Pi 上播放）

### 1、内录音频

首先需要解决内录音频的问题，默认情况下 FFmpeg 使用 ALSA 内录音频时无法获取到声卡设备，所以需要使用 ALSA Loopback 方式内录音频，录取声卡输出的音频而不影响正在播放的音频。

首先启用 `snd-aloop` 模块，编辑 `/etc/modules` 文件加入` snd-aloop` 模块跟随系统启动：

```
# sudo vim /etc/modules

# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.

i2c-dev
snd-aloop
```

然后在 Pi 用户的根目录下新建 `.asoundrc` 文件，填写如下内容：

```
# vim ~/.asoundrc

pcm.multi {
    type route;
    slave.pcm {
        type multi;
        slaves.a.pcm "output";
        slaves.b.pcm "loopin";
        slaves.a.channels 2;
        slaves.b.channels 2;
        bindings.0.slave a;
        bindings.0.channel 0;
        bindings.1.slave a;
        bindings.1.channel 1;
        bindings.2.slave b;
        bindings.2.channel 0;
        bindings.3.slave b;
        bindings.3.channel 1;
    }

    ttable.0.0 1;
    ttable.1.1 1;
    ttable.0.2 1;
    ttable.1.3 1;
}

pcm.!default {
	type plug
	slave.pcm "multi"
}

pcm.output {
	type hw
	card 0
}

pcm.loopin {
	type plug
	slave.pcm "hw:Loopback,0,0"
}

pcm.loopout {
	type plug
	slave.pcm "hw:Loopback,1,0"
}
```

完成后重启 Pi。

### 2、Raspotify

首先需要抱持 Pi 的系统处于最新状态：

```
sudo apt update
sudo apt upgrade
```

安装 `curl` 和 `apt-transport-https`：

```
sudo apt install -y apt-transport-https curl
```

接下来是安装 Raspotify GPG 密钥及项目仓库地址：

```
curl -sSL https://dtcooper.github.io/raspotify/key.asc | sudo apt-key add -v -
echo 'deb https://dtcooper.github.io/raspotify raspotify main' | sudo tee /etc/apt/sources.list.d/raspotify.list
```

现在可以将 Raspotify 安装到 Pi 了，别忘了更新一下仓库：

```
sudo apt update
sudo apt install raspotify
```

安装完成后，需要修改一下 Raspotify 的配置文件，注释掉几个选项，以及填入阁下的 Spotify 账户密码：

```
# sudo vim /etc/default/raspotify

# /etc/default/raspotify -- Arguments/configuration for librespot

# Device name on Spotify Connect
DEVICE_NAME="raspotify"

# Bitrate, one of 96 (low quality), 160 (default quality), or 320 (high quality)
BITRATE="320"

# Additional command line arguments for librespot can be set below.
# See `librespot -h` for more info. Make sure whatever arguments you specify
# aren't already covered by other variables in this file. (See the daemon's
# config at `/lib/systemd/system/raspotify.service` for more technical details.)
#
# To make your device visible on Spotify Connect across the Internet add your
# username and password which can be set via "Set device password", on your
# account settings, use `--username` and `--password`.
#
# To choose a different output device (ie a USB audio dongle or HDMI audio out),
# use `--device` with something like `--device hw:0,1`. Your mileage may vary.
#
OPTIONS="--username <USERNAME> --password <PASSWORD>"

# Uncomment to use a cache for downloaded audio files. Cache is disabled by
# default. It's best to leave this as-is if you want to use it, since
# permissions are properly set on the directory `/var/cache/raspotify'.
#CACHE_ARGS="--cache /var/cache/raspotify"

# By default, the volume normalization is enabled, add alternative volume
# arguments here if you'd like, but these should be fine.
VOLUME_ARGS="--enable-volume-normalisation --linear-volume --initial-volume=100"

# Backend could be set to pipe here, but it's for very advanced use cases of
# librespot, so you shouldn't need to change this under normal circumstances.
BACKEND_ARGS="--backend alsa"
```

重启 `sudo systemctl restart raspotify` 后既可在阁下的手机、电脑、平板等设备上的 Spotify “可用设备”中看到名为 `Raspotify` 的设备了，这个名字可以自行修改，在 `DEVICE_NAME="raspotify"` 选项中。选择该设备播放音乐后，既可在 Pi 上收听 Spotify ，但控制还是需要在手机或者电脑上控制。

### 3、FFmpeg 推流

Pi 默认自带了 FFmpeg，且功能上还算够用，至少对于推流来说该有的都有了。

简单粗暴：

```
ffmpeg -f x11grab -video_size 1920x1080  -framerate 5 -i :0.0 -f alsa -ac 2 -i hw:Loopback,1,0 -vcodec libx264 -preset ultrafast -b:v 2000k -maxrate 2000k -bufsize 2000k -acodec libmp3lame -ar 44100 -b:a 320k -f flv "rtmp://URL"
```

解释一下各个参数的意思（FFmpeg Wiki 上就有）：

- **-f x11grab** 使用 x11grab 屏幕录制
- **-video_size 1920x1080** 分辨率，这里使用 1080P
- **-framerate 5** 帧数，Pi 的白给性能帧数太高顶不住，以及录制屏幕基本是静态画面，所以楼主这里只用了 5 帧
- **-i :0.0** 录制屏幕区域
- **-f alsa** 使用 ALSA 进行录制音频
- **-ac 2** 双声道
- **-i hw:Loopback,1,0** 选择声卡设备，前面 `1、内录音频` 时有添加了 Loopback 虚拟声卡设备
- **-vcodec libx264** 视频编码器使用 libx264
- **-preset ultrafast** 这里预设值使用的是 `ultrafast `，既超快，还有 `superfast, veryfast, faster, fast, medium(default preset), slow, slower, veryslow, placebo` 等预设值，楼主使用 `ultrafast ` 勉强能够流畅，速率刚好达到 1x，如果是其他预设值的话基本都达不到 1x，也就是无法正常进行推流
- **-b:v 2000k -maxrate 2000k -bufsize 2000k** 这里是限制视频输出比特率，`-b:v` 是指定编码器使用的平均比特率，`-maxrate` 最大公差值，还有个 `-minrate` 最小公差值，`-bufsize` 缓冲区大小，这个的大小确定视频输出比特率的可变性，`-maxrate 和 -bufsize` 需要一起使用
- **-acodec libmp3lame** 音频编码器使用 libmp3lame
- **-ar 44100 -b:a 320k** 音频采样率为 44.1kHz，音频比特率为 320Kbps
- **-f flv** 输出为 flv 流媒体格式
- **"rtmp://URL"** 这条就是你的 rtmp 地址，例如 BiliBili，Youtube 直播等都会提供

如果阁下还需要添加说明参数，还请自行翻阅 FFMpeg Wiki，楼主仅使用到这么多。

### 总结

总的来说吧，吃灰的许久的 Pi 终于派上用场了，至于为什么不使用各位菊苣的开源项目进行推流呢？呃...楼主也不知道。

中间捣腾了好久，问题都出在 ALSA 内录上，能够录制视频但是无法录制音频，翻了不少文档和博客，最终还是在 FFMpeg Wiki 上找到了可行方案，看来没事儿还得多翻翻 Wiki。
