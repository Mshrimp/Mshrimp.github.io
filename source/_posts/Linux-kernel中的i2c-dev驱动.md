---
title: Linux-kernel中的i2c-dev驱动
date: 2020-04-23 21:43:49
tags: i2c
---





## Linux-kernel中的i2c-dev驱动

在i2c-dev.c文件中，实现了I2C适配器设备文件的功能，每个I2C适配器被分配一个设备节点；通过适配器访问设备文件节点，主设备号为89，次设备号为0～255；应用程序通过生成的设备节点/dev/i2c-X，使用open、close、read、write、ioctl系统调用进行访问；i2c-dev.c并不是针对特定的设备而设计；

<!--more-->



### 目录

[TOC]

### 简介












Linux内核的I2C子系统，支持I2C通用设备驱动；


用户态驱动：由应用层实现对硬件的控制；


代码实现在drivers/i2c/i2c-dev.c；

设备文件：/dev/i2c-X，X为序号；


















### 内核态实现






```
CONFIG_I2C_CHARDEV=y
```






```
// driver/i2c/Makefile
obj-$(CONFIG_I2C_CHARDEV)   += i2c-dev.o
```


















```c
// include/linux/i2c-dev.h
#define I2C_MAJOR   89      /* Device major number      */
```
















### 用户态实现




在Linux内核中，已经注册了的I2C适配器，可以在用户空间进行访问；i2c-dev驱动对每个I2C适配器生成一个设备节点/dev/i2c-X，主设备号固定为89，X为数字，是I2C适配器的编号，从0开始，编号和设备节点的此设备号一致；


```shell
# ls /dev/i2c-* -l
crw-rw----    1 root     root       89,   0 Nov  3 22:02 /dev/i2c-0
crw-rw----    1 root     root       89,   1 Nov  3 22:02 /dev/i2c-1
crw-rw----    1 root     root       89,   2 Nov  3 22:02 /dev/i2c-2
crw-rw----    1 root     root       89,   3 Nov  3 22:02 /dev/i2c-3
```


设备节点/dev/i2c-X的编号X和注册次序有关，使用前还是需要通过/sys/class/i2c-dev/来确定编号；


```shell
# ls /sys/class/i2c-dev/ -l
total 0
lrwxrwxrwx    1 root     root             0 Nov  4 04:01 i2c-0 -> ../../devices/platform/soc/2000000.i2c/i2c-0/i2c-dev/i2c-0
lrwxrwxrwx    1 root     root             0 Nov  4 04:01 i2c-1 -> ../../devices/platform/soc/2010000.i2c/i2c-1/i2c-dev/i2c-1
lrwxrwxrwx    1 root     root             0 Nov  4 04:01 i2c-2 -> ../../devices/platform/soc/2030000.i2c/i2c-2/i2c-dev/i2c-2
lrwxrwxrwx    1 root     root             0 Nov  4 04:01 i2c-3 -> ../../devices/platform/soc/2050000.i2c/i2c-3/i2c-dev/i2c-3
```


或者


```shell
# cat /sys/class/i2c-dev/i2c-0/name
2000000.i2c
# cat /sys/class/i2c-dev/i2c-2/name
2030000.i2c
```


这些设备节点实现了文件操作接口，用户空间通过这些I2C设备节点访问I2C适配器；对I2C设备进行读写时，可以通过调用read/write或ioctl来实现；在内核态read、write、ioctl系统调用都是通过i2c_transfer()函数实现和I2C设备的通信；




用户态使用open函数打开对应的I2C设备节点/dev/i2c-X，如：/dev/i2c-2；


```c
int fd = -1;
fd = open("/dev/i2c-2", O_RDWR);
```


i2c-dev为open的设备节点建立一个i2c_client；但是这个i2c_client并不加添加到i2c_adapter的client链表中，而是在用户关闭设备节点时，自动释放i2c_client；












#### read/write实现








```c
int i2c_write_bytes(int fd, unsigned short addr, unsigned char *data, int len)
{
    unsigned char *data_wr = NULL;
    int ret = -1;

    data_wr = malloc(len + 2);
    if (!data_wr) {
        printf("%s, malloc failed!\n", __func__);
        return -1;
    }

    data_wr[0] = addr / 0xff;
    data_wr[1] = addr % 0xff;

    memcpy(&data_wr[2], data, len);

    ioctl(fd, I2C_SLAVE, SLAVE_ADDR);
    ioctl(fd, I2C_TIMEOUT, 1);
    ioctl(fd, I2C_RETRIES, 1);

    ret = write(fd, data_wr, len+2);
    if (ret < 0) {
        printf("%s, write failed, ret: 0x%x\n", __func__, ret);
        return ret;
    }

    printf("%s, write ok, num: %d\n", __func__, ret);

    if (data_wr != NULL) {
        free(data_wr);
        data_wr = NULL;
    }

    return ret;
}
```






```c
int i2c_read_bytes(int fd, unsigned short addr, unsigned char *data, int len)
{
    unsigned char addr_slave[2] = { 0 };
    int ret = -1;

    ioctl(fd, I2C_SLAVE, SLAVE_ADDR);
    ioctl(fd, I2C_TIMEOUT, 1);
    ioctl(fd, I2C_RETRIES, 1);

    addr_slave[0] = addr / 0xff;
    addr_slave[1] = addr % 0xff;

    ret = write(fd, addr_slave, 2);
    if (ret < 0) {
        printf("%s, write failed, ret: 0x%x\n", __func__, ret);
        return ret;
    }

    ret = read(fd, data, len);
    if (ret < 0) {
        printf("%s, read failed, ret: 0x%x\n", __func__, ret);
        return ret;
    }

    printf("%s, read ok, num: %d\n", __func__, ret);

    return ret;
}
```












#### ioctl实现






```c
int i2c_write_bytes(int fd, unsigned short addr, unsigned char *data, int len)
{
    struct i2c_rdwr_ioctl_data data_wr;
    int ret = -1;

    data_wr.nmsgs = 1;
    data_wr.msgs = malloc(sizeof(struct i2c_msg) * data_wr.nmsgs);
    if (!data_wr.msgs) {
        printf("%s, msgs malloc failed!\n", __func__);
        return -1;
    }

    data_wr.msgs[0].addr = SLAVE_ADDR;
    data_wr.msgs[0].flags = 0;
    data_wr.msgs[0].len = len + 2;
    data_wr.msgs[0].buf = malloc(data_wr.msgs[0].len + 2);
    if (!data_wr.msgs[0].buf) {
        printf("%s, msgs buf malloc failed!\n", __func__);
        return -1;
    }
    data_wr.msgs[0].buf[0] = addr / 0xff;
    data_wr.msgs[0].buf[1] = addr % 0xff;

    memcpy(&data_wr.msgs[0].buf[2], data, len);

    ret = ioctl(fd, I2C_RDWR, (unsigned long)&data_wr);
    if (ret < 0) {
        printf("%s, ioctl failed, ret: 0x%x\n", __func__, ret);
        return ret;
    }

    printf("%s, write ok, num: %d\n", __func__, ret);

    if (data_wr.msgs[0].buf != NULL) {
        free(data_wr.msgs[0].buf);
        data_wr.msgs[0].buf = NULL;
    }

    if (data_wr.msgs != NULL) {
        free(data_wr.msgs);
        data_wr.msgs = NULL;
    }

    return ret;
}
```












```c
int i2c_read_bytes(int fd, unsigned short addr, unsigned char *data, int len)
{
    struct i2c_rdwr_ioctl_data data_rd;
    int ret = -1;

    data_rd.nmsgs = 2;
    data_rd.msgs = malloc(sizeof(struct i2c_msg) * data_rd.nmsgs);
    if (!data_rd.msgs) {
        printf("%s, msgs malloc failed!\n", __func__);
        return -1;
    }

    data_rd.msgs[0].addr = SLAVE_ADDR;
    data_rd.msgs[0].flags = 0;
    data_rd.msgs[0].len = 2;
    data_rd.msgs[0].buf = malloc(data_rd.msgs[0].len);
    if (!data_rd.msgs[0].buf) {
        printf("%s, msgs buf malloc failed!\n", __func__);
        return -1;
    }
    data_rd.msgs[0].buf[0] = addr / 0xff;
    data_rd.msgs[0].buf[1] = addr % 0xff;

    data_rd.msgs[1].addr = SLAVE_ADDR;
    data_rd.msgs[1].flags = I2C_M_RD;
    data_rd.msgs[1].len = len;
    data_rd.msgs[1].buf = malloc(data_rd.msgs[0].len);
    if (!data_rd.msgs[0].buf) {
        printf("%s, msgs buf malloc failed!\n", __func__);
        return -1;
    }
    memset(data_rd.msgs[1].buf, 0, data_rd.msgs[1].len);

    ret = ioctl(fd, I2C_RDWR, (unsigned long)&data_rd);
    if (ret < 0) {
        printf("%s, ioctl failed, ret: 0x%x\n", __func__, ret);
        return ret;
    }
    memcpy(data, data_rd.msgs[1].buf, len);

    printf("%s, read ok, num: %d\n", __func__, ret);

    if (data_rd.msgs[0].buf != NULL) {
        free(data_rd.msgs[0].buf);
        data_rd.msgs[0].buf = NULL;
    }

    if (data_rd.msgs != NULL) {
        free(data_rd.msgs);
        data_rd.msgs = NULL;
    }

    return ret;
}
```




















#### main函数




```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <linux/i2c.h>
#include <linux/i2c-dev.h>

#define SLAVE_ADDR  0x51

int arr_show(unsigned char *data, int len)
{
    int i = 0;

    for (i = 0; i < len; i++) {
        printf("data[%d]: 0x%x\n", i, data[i]);
    }

    return 0;
}

void usage(void)
{
    printf("xxx -r addr len\n");
    printf("xxx -w addr data1 data2 ...\n");
}

int main(int argc, char *argv[])
{
    int opt;
    int fd = -1;

    unsigned short addr;
    unsigned char buf[256] = { 0 };
    int len = 0;
    int i = 0;

    if (argc < 4) {
        usage();
        return -1;
    }

    fd = open("/dev/i2c-2", O_RDWR);
    if (fd < 0) {
        printf("%s, open failed!\n", __func__);
        return -1;
    }

    while ((opt = getopt(argc, argv, "w:r:")) != -1) {
        printf("optarg: %s\n", optarg);
        printf("optind: %d\n", optind);
        printf("argc: %d\n", argc);
        printf("argv[optind]: %s\n", argv[optind]);

        addr = (unsigned short)strtol(optarg, NULL, 0);
        printf("addr: %d\n", addr);
        switch(opt) {
            case 'w':
                for (len = 0; optind < argc; optind++, len++) {
                    buf[len] = (unsigned char)strtol(argv[optind], NULL, 0);
                }
                printf("len: %d\n", len);

                i2c_write_bytes(fd, addr, buf, len);
                break;
            case 'r':
                len = (unsigned int)strtol(argv[optind], NULL, 0);
                printf("len: %d\n", len);

                i2c_read_bytes(fd, addr, buf, len);

                arr_show(buf, len);
                break;
            default:
                printf("Invalid parameter!\n");
                usage;
                break;
        }
    }
    close(fd);

    return 0;
}
```





















