---
title: 视频转码时保留元数据
date: '2021-12-29T00:00:00+08:00'
url: /post/y-xflXZFC/
tags:
- exif
- media
feature: https://res.cloudinary.com/codenote-net/image/upload/w_auto:100:582,f_auto,q_auto/v1555944471/ExifTool.png
---

最近有几个旧相机拍摄的视频需要转码，但转码后发现相机生产商、相机型号、视频创建日期等元数据丢失。这篇文章记录了一次用 exiftool 保留视频文件元数据的实践。

转码的需求主要来源是，我需要用 digikam 来管理照片和视频，而这些视频的编码格式不受软件支持。这些视频是用 PENTAX Optio H90 拍摄的，封装格式为 AVI。

保留元数据的目的是，我希望在 digikam 中能通过拍摄时间、相机型号对照片和视频排序。

注：下列 shell 环境主要为 Powershell。

## 1 源文件元数据分析

查看视频文件的元数据

```
exiftool -G1 -s -a
```

## 1.1 设备型号

- 名为 `MakerNotes` 的元数据节中记录了 `Make` 和 `Model`。但这似乎是设备厂商自定义的元数据节。
- 标准的 `EXIF` 元数据节中没有记录设备型号。因此即便是源文件，也无法被主流软件读出设备型号。

## 1.2 拍摄时间

- 仅有名为 `RIFF:DateTimeOriginal` 的元数据标签记录了视频的拍摄时间。

## 2 转换结果分析

使用 Adobe Media Encoder 转码后文件的元数据情况如下：

- 设备型号。`MakerNotes` 全部丢失，`Make` 丢失，`Model` 丢失；
- 拍摄时间。`RIFF:DateTimeOriginal` 被存储至 `XMP-exif:DateTimeOriginal`，但在 macOS 访达、digikam 中，该元数据无法被识别为拍摄时间；
- 其他时间。新增的 `QuickTime:CreateDate`、`QuickTime:ModifyDate`、`XMP-xmp:CreateDate`、`XMP-xmp:ModifyDate` 元数据标签均为转码时间，即新文件创建时间，而非拍摄时间。

关于拍摄时间的问题，经过了解，一般认为，`XMP-exif` 不是标准元数据节，`xmp-tiff` 更为通用。

## 3 解决问题

目标很明确，要在主要应用程序中正确读取到上述数据。

## 3.1 增加设备型号元数据

经过测试，`xmp-tiff` 是设备型号数据常用的元数据节，可以被访达、digikam 读取。根据前面的分析，由于转码后设备型号被丢弃，这里需要对转码后的视频文件进行手动写入元数据。对同一批旧视频、器材一致的情况，需要写入的内容是固定的，比较好操作。

```
# 单个文件的处理 exiftool -Make='PENTAX' -Model='PENTAX Optio H90' IMGP0982.mp4
# 批量文件的处理，Winodws exiftool -Make='PENTAX' -Model='PENTAX Optio H90' mp4
```

上述命令实现的结果是，将 `Make` 和 `Model` 写入到目标文件 `xmp-tiff` 元数据节下。

## 3.2 复制拍摄时间元数据

经过测试，`XMP-xmp`、`QuickTime` 是拍摄时间数据常用的元数据节。根据前面的分析，由于拍摄时间未丢失，这里需要依据每一个目标文件的 `XMP-exif:DateTimeOriginal` 来更新 `XMP-xmp`、`QuickTime` 等元数据节中的时间标签。

```
# 单个文件的处理 exiftool "-AllDates<XMP-exif:DateTimeOriginal" IMGP0982.mp4
# 批量文件的处理，macOS exiftool "-AllDates<XMP-exif:DateTimeOriginal" mp4
```

执行后，目标文件的日期时间相关元数据标签被改写，包括：

- `QuickTime:CreateDate`
- `QuickTime:ModifyDate`
- `XMP-xmp:CreateDate`
- `XMP-xmp:ModifyDate`
- `UserData:DateTimeOriginal`

至于 `ModifyDate`，我个人倾向于认为是媒体文件被修改的时间，例如截取、裁剪、信息修改等，虽然也被调整为拍摄时间，但拍摄时间比较重要，也就无所谓了。

## 参考

- [exiftool shortcuts tags](https://exiftool.org/TagNames/Shortcuts.html)