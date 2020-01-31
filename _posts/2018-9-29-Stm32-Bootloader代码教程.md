---
layout:     post
title:      Stm32 Bootloader代码教程
subtitle:   我的stm32 Bootloader食用指南
date:       2018-9-18
author:     Hehesheng
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - stm32
    - 开发技巧
    - arm结构
---

## Stm32 Bootloader代码教程

### 前言

这个程序是在8月末写好的, 至于拖到现在才写教程和说明就是因为我懒(理不直气也壮).

### 使用库

- stm32官方标准库
- FatFs R0.13b

### 原理

原理个人感觉没什么好说的, 就是使用FatFs读取SD卡, 找到Bin文件, 然后进行IAP升级. 写下教程也是写一下个人折腾经验.

### 主函数

函数主体截图:

![image-20180929152251515](http://pfr80kuto.bkt.clouddn.com/img/2018/9/image-20180929152251515.png)

关键函数是: 17行, 23行, 和32行的三个函数

```c
//17行的findBin函数
u8 findBin(const TCHAR *path, TCHAR *binName)
{
    DIR dir;
    FRESULT res;
    FILINFO fileinfo;

    res = f_opendir(&dir, (const TCHAR *)path); //打开一个目录
    if (res == FR_OK)
    {
        while (1)
        {
            res = f_readdir(&dir, &fileinfo); //读取目录下的一个文件
            if (res != FR_OK || fileinfo.fname[0] == 0)
                break;                     //错误了/到末尾了,退出
            if (fileinfo.fattrib & AM_ARC) //如果读到这是一个文件
            {
                if ((!strcmp(fileinfo.fname + (strlen(fileinfo.fname) - 4), ".bin") |
                     !strcmp(fileinfo.fname + (strlen(fileinfo.fname) - 4), ".BIN"))) //这里是在进行验证的时候发现有的文件电脑上是小写, 单片机读出来却是大写,比较迷得一个Bug
                {
                    strcpy(binName, fileinfo.fname);
                    f_closedir(&dir);
                    return 1; //成功找到文件
                }
            }
        }
    }
    f_closedir(&dir);
    return 0;
}
```

找到文件就是写flash了:

```c
//23行的updateApplication函数, 每次往flash中写入2k
FRESULT updateApplication(const TCHAR *path)
{
    FIL fsrc;
    FRESULT fr;
    UINT br;
    BYTE buffer[2048]; //2k缓存
    uint32_t address = ApplicationAddress;
    fr = f_open(&fsrc, path,eFA_READ);
    if (fr)
        return fr;
    while (1)
    {
        fr = f_read(&fsrc, buffer, sizeof buffer, &br);
        if (fr || br == 0)
            break;
        if (flashWrite(address, (uint32_t *)buffer, br) != FLASH_COMPLETE)
            return FR_DISK_ERR;
        USART_SendData(USART1, '#');
        address += br;
    }
    return FR_OK;
}
```

最后是IAP跳转

```c
//32行的jumpToApp函数, 要记得重置栈顶地址, 可以使用固件库自带的函数__set_MSP
void jumpToApp(void)
{
    unsigned int JumpAddress;
    typedef void (*pFunction)(void);
    pFunction Jump_To_Application;
    //程序的第二个字存放的是复位函数的地址，
    //通过这个地址跳转的应用程序中
    JumpAddress = *(__IO uint32_t *)(ApplicationAddress + 4);

    Jump_To_Application = (pFunction)JumpAddress;

    //指向应用程序的栈顶
    __set_MSP(*(__IO uint32_t *)ApplicationAddress);

    //执行应用程序的复位函数
    beforeJumpToApp();
    Jump_To_Application();
}
```

这里提一下, 由于试了一下将终端向量表在Bootloader中设置的话, 必须要将App中`system_Init`函数中设置的中断向量表给注释掉, emmmm, 至于这个问题怎么解决, 我想到的是使用`__weak`来吧原`system_Init`标识一下, 自己再重写一个能识别是BootLoader还是App的Init函数, 虽然方便, 但是太折腾了, 我选择死亡, 所以写了一个简单的函数, 简单封装一下:

```c
void beforeAppBegin(void)
{
    NVIC_SetVectorTable(NVIC_VectTab_FLASH, ApplicationAddress^BootloaderAddress); //设置中断向量表
}
```

App的main函数开头调用一下.

> Talk is cheap, show me the code.

代码我上传到我的[GitHub](https://github.com/Hehesheng/stm32f4_bootloader)仓库了, 大家来有兴趣欢迎fork和clone. :)


