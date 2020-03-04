---
title: Android 目录
date: 2020-03-02
tag: Android

---

[Toc]

# Android 目录

## 内部存储

内部存储位于系统中很特殊的一个位置，对于设备中每一个安装的 App，系统都会在 **data/data/packagename/xxx**或者**data/user/0/packagename/xxx** 自动创建与之对应的文件夹。如果你想将文件存储于内部存储中，那么文件默认只能被你的应用访问到，且一个应用所创建的所有文件都在和应用包名相同的目录下。也就是说应用创建于内部存储的文件，与这个应用是关联起来的。当一个应用卸载之后，内部存储中的这些文件也被删除。对于这个内部目录，用户是无法访问的，除非获取root权限。

- `context.getFilesDir()`：`/data/user/0/packagename/files`
- `context.getCacheDir()`：`/data/user/0/packagename/cache`缓存目录，当内存不足时会优先被删掉

## 外部存储

### 机身自带的外部存储

这种属于现在主流的一种情况，大部分手机自带的存储空间变得很大，不再需要SD卡

私有包目录： `/storage/emulated/0/Android/data/packagename/`

- `context.getExternalCacheDir()`：`/storage/emulated/0/Android/data/packagename/cache`

- `context.getExternalFilesDir(String)`：`/storage/emulated/0/Android/data/packagename/files/`

    |              API              |  **对应路径**   |
    | :---------------------------: | :-------------: |
    | `Environment.DIRECTORY_MUSIC` |     `Music`     |
    |     `DIRECTORY_PODCASTS`      |   `Podcasts`    |
    |     `DIRECTORY_RINGTONES`     |   `Ringtones`   |
    |      `DIRECTORY_ALARMS`       |    `Alarms`     |
    |   `DIRECTORY_NOTIFICATIONS`   | `Notifications` |
    |     `DIRECTORY_PICTURES`      |   `Pictures`    |
    |      `DIRECTORY_MOVIES`       |    `Movies`     |
    |     `DIRECTORY_DOWNLOADS`     |   `Download`    |
    |       `DIRECTORY_DCIM`        |     `DCIM`      |
    |     `DIRECTORY_DOCUMENTS`     |   `Documents`   |
    |    `DIRECTORY_SCREENSHOTS`    |  `Screenshots`  |
    |    `DIRECTORY_AUDIOBOOKS`     |  `Audiobooks`   |

    可根据`Environment`获取包目录下对应的目录

公有目录：`/storage/emulated/0/`

- `Environment.getExternalStoragePublicDirectory(String)`：`/storage/emulated/0/`

    通过`Environment`访问公有目录，如Music：`Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_MUSIC)`对应的目录是`/storage/emulated/0/Music`，其他如Download等目录对应

    *在Android Q之后由于权限改变，不建议通过Environment去访问公有的目录，而是在自己的包目录下建立对应的目录*

### 扩展SD卡的外部存储

就是通过SD卡扩展的存储空间，一般来说是可以自由访问的空间，在以前的Android版本中，SD就是前面所说的外部存储来使用的

SD卡的状态

|           标识            |                   状态                   |
| :-----------------------: | :--------------------------------------: |
|      `MEDIA_UNKNOWN`      |                 SD卡未知                 |
|      `MEDIA_REMOVED`      |                 SD卡移除                 |
|     `MEDIA_UNMOUNTED`     |                SD卡未安装                |
|     `MEDIA_CHECKING`      |         SD卡检查中，刚装上SD卡时         |
|       `MEDIA_NOFS`        |  SD卡为空白或正在使用不受支持的文件系统  |
|      `MEDIA_MOUNTED`      |                 SD卡安装                 |
| `MEDIA_MOUNTED_READ_ONLY` |             SD卡安装但是只读             |
|      `MEDIA_SHARED`       |                 SD卡共享                 |
|    `MEDIA_BAD_REMOVAL`    |               SD卡移除错误               |
|    `MEDIA_UNMOUNTABLE`    | 存在SD卡但是不能挂载，例如发生在介质损坏 |
|     `MEDIA_EJECTING`      |                 SD卡弹出                 |

访问SD卡通常会通过`Environment`来进行访问

- `Environment.getExternalStorageDirectory()`：SD卡根目录

## 系统目录

- `getRootDirectory()`：`/system`
- `getDataDirectory()`：`/data`
- `getDownloadCacheDirectory`：`/cache`

## 参考

- [Android基础   你必须了解的应用文件目录](https://juejin.im/post/5a39e33bf265da430b7b60fc)
- [一篇文章搞懂android存储目录结构](https://juejin.im/post/5de7772af265da3398561133)