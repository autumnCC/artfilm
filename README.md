README.md
ArtFilm (MuseFilm Mac) — 原生胶卷照片管理应用 / Native Film Photo Manager
🎞️ 归档胶卷，赛博观片 — 专为 Mac 设计的原生胶卷照片管理工具
Archive your film rolls, view them on a cyber light table — a native macOS app for film photographers.

版本 / Version: 1.1.0

📋 目录 / Table of Contents
v1.1.0 更新总览 / What's New
修复点详解 / Fix Details
新增功能调用方法 / New Feature Usage
本地化说明 / Localization
胶卷图标系统 / Film Icon System
项目结构 / Project Structure
安装与编译 / Build & Install
常见问题 / FAQ
🆕 v1.1.0 更新总览 / What's New
本次更新全面重构了照片导入底层架构，修复了全部已知导入故障，并新增本地化、胶卷图标等特性。

This update completely refactors the photo import architecture, fixes all known import failures, and adds localization + film icon features.

#	修复/新增 / Fix & Feature	说明 / Description
1	异步多线程导入 / Async multithreaded import	TaskGroup 并发导入，不再阻塞主线程 / Concurrent import via TaskGroup, no more main-thread blocking
2	断点续传 / Checkpoint & resume	支持暂停/恢复导入，外接磁盘断连自动暂停 / Pause/resume support; auto-pause on disk disconnect
3	单张失败不阻断整批 / Single failure isolation	每张照片独立 try/catch，一张失败不影响其余 / Per-photo error isolation; one failure doesn't block the batch
4	完整错误捕获 / Complete error capture	ImportFailure 记录文件名/原因/时间，失败清单可视化 / Detailed failure records with visual failure list
5	0KB/损坏文件过滤 / 0KB & corrupt file filtering	自动跳过 0KB 文件和无法解码的损坏图片 / Auto-skip empty and corrupt files
6	非法文件名转义 / Filename escaping	转义 / : * ? " < > | 等路径非法字符 / Escape illegal path characters
7	EXIF 解析 / EXIF parsing	解析拍摄日期、相机型号 / Parse capture date and camera model
8	TIFF/RAW 兼容 / TIFF & RAW support	统一 CGImageSource 解码，支持 TIFF/CR2/CR3/NEF/ARW/DNG 等 / Unified CGImageSource decoder for TIFF + RAW formats
9	权限自动检测 / Permission auto-detection	无磁盘访问权限时弹出系统引导弹窗 / System guidance dialog when disk access is denied
10	磁盘挂载监听 / Disk mount monitoring	外接 SD/移动硬盘断连自动暂停并弹窗提醒 / Auto-pause + alert on external disk disconnect
11	缩略图内存修复 / Thumbnail memory fix	CGImageSource 下采样，修复高分辨率底片内存泄漏 / CGImageSource downsampling fixes memory leaks
12	实时进度条 / Real-time progress bar	按张更新进度，不再 0→100 跳变 / Per-photo progress, no more 0→100 jumps
13	失败日志导出 / Failure log export	导出 TSV 格式失败日志到 Finder / Export TSV failure log to Finder
14	全界面本地化 / Full UI localization	中文系统中文，非中文系统英文 / Chinese on Chinese systems, English otherwise
15	胶卷图标系统 / Film icon system	26 种默认胶卷对应品牌图标 + 自定义图标修改 / 26 default brand icons + custom icon support
🔧 修复点详解 / Fix Details
1. 导入失败底层原因排查与修复 / Root Cause Analysis & Fixes
问题 / Problem: 旧版 importPhotos 在主线程同步执行，单张失败仅 print 不记录，进度 0→100 跳变，高分辨率 TIFF 导入时内存暴涨崩溃。

修复 / Fix:

ImportManager.swift — 全新异步导入引擎
StorageManager.swift — 暴露 appSupportPhotosURL，新增 addPhotoThreadSafe 线程安全写入
使用 AsyncStream<ImportProgress> 实时推送每张导入结果
// 调用示例 / Usage
for await progress in importManager.importPhotos(from: urls, toRoll: rollId) {
    // progress.completed / progress.total / progress.failures
}
2. 缩略图内存泄漏修复 / Thumbnail Memory Leak Fix
问题 / Problem: 旧版 generateThumbnail 用 NSImage(contentsOf:) + lockFocus() 将整张高分辨率底片（可达 100MB+）解压进内存，批量导入时内存峰值可达数 GB，导致 OOM 崩溃。

修复 / Fix: 改用 CGImageSource + kCGImageSourceThumbnailMaxPixelSize 直接下采样，仅生成 600px 缩略图，峰值内存 < 5MB。配合 autoreleasepool 释放中间对象。

// ImportManager.generateThumbnailMemorySafe()
let options: [CFString: Any] = [
    kCGImageSourceCreateThumbnailFromImageAlways: true,
    kCGImageSourceThumbnailMaxPixelSize: 600,
    kCGImageSourceShouldCache: false  // 不缓存原图
]
let cgImage = CGImageSourceCreateThumbnailAtIndex(source, 0, options as CFDictionary)
PhotoThumbView（DashboardView.swift）同样改为后台 CGImageSource 下采样加载，避免网格视图卡顿。

3. 权限检测与引导 / Permission Detection & Guidance
PermissionManager.swift 在选择文件夹时自动检测访问权限：

checkFolderAccess(_:) — 枚举文件夹内容判断可读性
showPermissionDialog() — 弹出三选项弹窗（打开系统设置 / 重新选择 / 取消）
openSystemPrivacySettings() — 跳转系统设置「隐私与安全性」
4. 磁盘断连监听 / Disk Disconnect Monitoring
ImportManager 监听 NSWorkspace.didUnmountVolumeNotification，当导入源所在卷断开时：
