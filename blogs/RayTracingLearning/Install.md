## **How to Get Started with DirectX Raytracing?**

*   An installation of [Windows 10 RS4 (v.1803)](https://en.wikipedia.org/wiki/Windows_10_version_history) or higher

```c
DirectX Raytracing is experimental on RS4, so you need to enable Developer Mode.
On Windows 10 RS5, DXR should be integrated which will simplify use.
```

*   You will need GPU hardware and software drivers that support DXR.

```
NVIDIA supports DXR on Volta GPUs with drivers 396.x and above.
NVIDIA supports DXR on Turing GPUs with drivers in the 400 series.
All DirectX 12 class GPUs can run DXR via Microsoft's fallback layer
On Windows 10 RS4, this fallback uses a subtly different API, so not all tutorials support both hardware and fallback modes.
```

*   For development on RS4, you need Microsoft's [DXR installation package](http://forums.directxtech.com/index.php?topic=5860.0)

```c
Many of the tutorials above come with this package.
On RS5, this may be integrated with the standard Windows SDK
```





[Windows版本查看](https://en.wikipedia.org/wiki/Windows_10_version_history)，我们一般看到的是build版本。

Win+R ，输入cmd在命令行最上方会显示，其中17763就是Build版本，可以看表得到对应的Version：

```c
Microsoft Windows [版本 10.0.17763.1577]
(c) 2018 Microsoft Corporation。保留所有权利。

C:\Users\User>
```






|                           Version                            |                           Codename                           |    Marketing name    |             Build             |   Release date    |
| :----------------------------------------------------------: | :----------------------------------------------------------: | :------------------: | :---------------------------: | :---------------: |
|          HomePro Pro Education Pro for Workstations          |                     Enterprise Education                     |         LTSC         |            Mobile             |                   |
|                             1507                             | [Threshold 1](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_1507)) |         N/A          |             10240             |   July 29, 2015   |
|                             1511                             | [Threshold 2](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_1511)) |   November Update    |             10586             | November 10, 2015 |
|                             1607                             | [Redstone 1](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_1607)) |  Anniversary Update  |             14393             |  August 2, 2016   |
|                             1703                             | [Redstone 2](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_1703)) |   Creators Update    |             15063             |   April 5, 2017   |
|                             1709                             | [Redstone 3](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_1709)) | Fall Creators Update |             16299             | October 17, 2017  |
|                             1803                             | [Redstone 4](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_1803)) |  April 2018 Update   |             17134             |  April 30, 2018   |
|                             1809                             | [Redstone 5](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_1809)) | October 2018 Update  |             17763             | November 13, 2018 |
|                             1903                             | [19H1](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_1903)) |   May 2019 Update    |             18362             |   May 21, 2019    |
|                             1909                             | [19H2](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_1909)) | November 2019 Update |             18363             | November 12, 2019 |
|                             2004                             | [20H1](https://en.wikipedia.org/wiki/Windows_10_version_history_(version_2004)) |   May 2020 Update    |             19041             |   May 27, 2020    |
|                             20H2                             | [20H2](https://en.wikipedia.org/wiki/Windows_10_version_history#Version_20H2_(October_2020_Update)) | October 2020 Update  |             19042             | October 20, 2020  |
| [Dev Channel](https://en.wikipedia.org/wiki/Windows_10_version_history#Dev_Channel) |                            20270                             |         N/A          | Rolling Builds in Development |                   |