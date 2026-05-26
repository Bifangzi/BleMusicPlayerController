# ESP32-P4C6 Music Player

基于 uni-app 开发的 ESP32-P4C6 蓝牙音乐播放器控制端。项目用于通过手机 APP 连接 ESP32C6 BLE 模块，并将控制指令发送至 ESP32P4 音乐播放器主控，实现音乐播放状态显示、播放控制、音量调节、蓝牙通信调试以及项目信息展示等功能。

本 README 根据当前工程中的 `index.vue` 页面代码整理，适合直接放置在项目根目录作为仓库说明文档。

## 项目简介

本项目面向基于 ESP32P4 与 ESP32C6 的嵌入式音乐播放器系统。手机端 APP 通过 BLE 与 ESP32C6 建立连接，ESP32C6 作为蓝牙通信桥接端，将手机侧发送的控制信息转发至 ESP32P4；ESP32P4 负责音乐播放、音频处理与状态上报。APP 端主要承担设备连接、指令发送、状态显示和交互控制等任务。

当前页面已经实现 BLE 设备扫描、连接、服务与特征值选择、Notify 监听、播放器状态解析、播放器控制按钮、底部开发者信息与 GitHub 开源仓库跳转等功能。

## 功能特性

### 1. BLE 蓝牙连接

- 初始化手机蓝牙适配器。
- 扫描附近 BLE 设备并在底部弹窗中显示设备列表。
- 点击设备后自动建立 BLE 连接。
- 连接成功后自动停止蓝牙搜索，减少系统资源消耗。
- 自动读取设备服务和特征值。
- 支持手动选择服务、写入特征值和监听特征值。
- 支持开启和停止 Notify 监听。
- 支持断开当前 BLE 连接。

### 2. 音乐播放器界面

- 显示当前播放音乐名称。
- 显示播放状态，包括“播放中”和“已暂停”。
- 显示当前音量。
- 显示当前播放时间和歌曲总时间。
- 使用进度条展示播放进度。
- 支持封面图片配置，未配置时显示默认音乐符号。

### 3. 播放控制按钮

页面提供五个播放器控制按钮，默认命令如下：

| 按钮 | 默认命令 | 功能含义 |
| --- | --- | --- |
| 音量减 | `vol+down` | 降低音量 |
| 上一曲 | `music+up` | 切换到上一首音乐 |
| 播放/暂停 | `music+play` | 切换播放状态 |
| 下一曲 | `music+down` | 切换到下一首音乐 |
| 音量加 | `vol+up` | 提高音量 |

按钮命令通过 `playerCommandMap` 统一管理，后续如果 ESP32P4 端的控制协议发生变化，只需要修改该映射关系即可。

### 4. 播放状态上报

APP 端通过 BLE Notify 接收设备端上报的数据。当前代码支持解析 `music_status` 类型的 JSON 数据，并自动刷新播放器界面。

设备端建议按行发送 JSON 字符串，即每条状态数据以换行符 `\n` 结束。例如：

```json
{
  "type": "music_status",
  "ready": true,
  "state": "playing",
  "name": "Song Name - Artist",
  "volume": 55,
  "play_ms": 65000,
  "total_ms": 240000,
  "play_time": "01:05",
  "total_time": "04:00",
  "time_text": "01:05/04:00",
  "progress": 27,
  "index": 1,
  "count": 10
}
```

其中 `state` 可使用 `playing` 或 `paused`，`progress` 建议为 0 到 100 之间的数值。

### 5. 底部项目信息栏

底部信息栏用于展示项目和开发者相关信息，包括：

- 项目名称或开发者名称。
- 开发者角色描述。
- 项目简介。
- 版本号。
- 编译时间。
- 硬件平台。
- 通信协议。
- GitHub 开源仓库入口。

GitHub 开源入口支持点击跳转。H5 端会打开新窗口，App 端会调用系统浏览器，其他平台会复制仓库地址到剪贴板。

## 技术栈

- uni-app
- Vue 单文件组件
- uni-popup 弹窗组件
- BLE 蓝牙低功耗通信
- ESP32C6 BLE UART
- ESP32P4 音乐播放器主控

## BLE 通信说明

### 默认服务与特征值 UUID

当前工程默认使用 Nordic UART Service 风格的 BLE UUID：

```js
// 服务 UUID
defaultServiceId: '6E400001-B5A3-F393-E0A9-E50E24DCCA9E'

// 手机写入设备端 RX 特征值 UUID
defaultWriteCharacteristicId: '6E400002-B5A3-F393-E0A9-E50E24DCCA9E'

// 设备端 Notify 返回 TX 特征值 UUID
defaultNotifyCharacteristicId: '6E400003-B5A3-F393-E0A9-E50E24DCCA9E'
```

如果设备端使用其他 BLE 服务或特征值，可以直接修改上述默认值，也可以在 APP 界面中手动选择服务、写入对象和监听对象。

### 发送数据格式

APP 端发送字符串命令，内部会将字符串转换为 UTF-8 编码的 `ArrayBuffer`，并通过 `uni.writeBLECharacteristicValue` 写入 BLE 特征值。

为了兼容 BLE 单包长度限制，代码会将待发送数据按 20 字节分片发送，每片之间保留短暂延迟。

### 接收数据格式

APP 端通过 `uni.onBLECharacteristicValueChange` 接收 Notify 数据，并完成以下处理：

1. 将 `ArrayBuffer` 转换为字符串。
2. 将收到的数据追加到缓存区。
3. 按换行符拆分完整消息。
4. 尝试将每一行解析为 JSON。
5. 当 JSON 的 `type` 为 `music_status` 时刷新播放器状态。

## 目录建议

当前上传文件主要为页面文件，可按常见 uni-app 项目结构放置：

```text
project-root/
├── pages/
│   └── index/
│       └── index.vue
├── static/
│   ├── avatar-128.jpg
│   └── player-icons/
├── uni_modules/
│   └── uni-popup/
├── App.vue
├── main.js
├── manifest.json
├── pages.json
└── README.md
```

如果不使用头像图片，可以将 `developerInfo.avatar` 留空，页面会自动显示名称首字母占位图。

## 快速开始

### 1. 导入工程

使用 HBuilderX 或支持 uni-app 的开发环境导入项目。

### 2. 检查页面路径

确认页面文件位于 `pages/index/index.vue`，并在 `pages.json` 中正确配置页面路径。

### 3. 检查组件依赖

当前页面使用了 `uni-popup` 组件。若项目中没有该组件，请先在插件市场或 `uni_modules` 中引入 `uni-popup`。

### 4. 配置蓝牙权限

在打包到 App 平台前，需要确认 `manifest.json` 中已经配置蓝牙相关权限。Android 平台通常需要蓝牙、定位以及 Android 12 及以上版本相关的蓝牙扫描和连接权限。

### 5. 运行项目

可以先运行到 H5 或手机 App 进行界面预览。由于 BLE 能力主要依赖真机环境，建议使用真机调试蓝牙连接、搜索、写入和 Notify 监听功能。

## 常用配置

### 修改 BLE 默认 UUID

在 `data()` 中修改以下字段：

```js
defaultServiceId: '你的服务UUID',
defaultWriteCharacteristicId: '你的写入特征值UUID',
defaultNotifyCharacteristicId: '你的Notify特征值UUID'
```

### 修改播放器控制命令

可以直接修改 `playerCommandMap`：

```js
playerCommandMap: {
  volDown: 'vol+down',
  prev: 'music+up',
  playPause: 'music+play',
  next: 'music+down',
  volUp: 'vol+up'
}
```

也可以通过页面暴露的接口动态设置：

```js
this.SetPlayerCommandMap({
  playPause: 'your_play_command',
  next: 'your_next_command'
})
```

### 修改播放器按钮图标

默认按钮在未配置图片时使用文字图标。如果需要使用图片，可以修改 `playerButtonList` 中的 `icon`、`activeIcon` 或 `inactiveIcon` 字段，也可以调用：

```js
this.SetPlayerButtonAssets({
  playPause: {
    activeIcon: '/static/icons/pause.png',
    inactiveIcon: '/static/icons/play.png'
  },
  next: {
    icon: '/static/icons/next.png'
  }
})
```

### 修改底部信息

底部开发者和项目信息可通过 `SetFooterInfo`、`SetDeveloperInfo` 和 `SetAppInfo` 修改。

```js
this.SetFooterInfo({
  developerInfo: {
    avatar: '/static/avatar-128.jpg',
    name: 'ESP32-P4C6 Music Player',
    role: 'Developed by BI9BCY',
    description: '基于 ESP32P4 与 ESP32C6 BLE UART 的音乐播放器控制端'
  },
  appInfo: {
    version: 'v2.1.3',
    buildTime: '2026-05-27',
    firmware: 'ESP32-P4C6',
    protocol: 'BLE Notify / UART JSON',
    repositoryUrl: 'https://github.com/Bifangzi/BleMusicPlayerController',
    openSourceTitle: 'Open Source on GitHub',
    openSourceDesc: '本项目基于MIT协议开源，点击查看源码仓库'
  }
})
```

## 与 ESP32 端的配合建议

ESP32C6 端建议提供 BLE UART 服务，并至少包含：

- 一个可写特征值，用于接收手机端控制命令
- 一个 Notify 或 Indicate 特征值，用于向手机端上报播放器状态

ESP32P4 端建议在音乐状态变化时主动上报 `music_status` 数据，例如：

- 播放或暂停状态变化。
- 音量变化
- 当前播放歌曲变化
- 播放时间周期性更新
- 播放列表索引变化

为了保证 APP 端能够正确拆包解析，建议每条 JSON 状态信息以 `\n` 结尾

## 常见问题

### 1. 搜索不到 BLE 设备

请检查手机蓝牙和定位权限是否开启，并确认 ESP32C6 正在广播 BLE 服务。Android 设备还需要注意系统版本对应的蓝牙扫描权限

### 2. 可以连接但不能发送命令

请检查当前选择的写入特征值是否具有 `write` 或 `writeNoResponse` 属性。如果默认 UUID 与设备端不一致，可以在界面中手动切换服务和写入对象

### 3. 可以连接但收不到状态

请检查当前监听对象是否具有 `notify` 或 `indicate` 属性，并确认设备端已经主动发送 Notify 数据

### 4. 播放状态不刷新

请确认设备端发送的是合法 JSON，并且 `type` 字段为 `music_status`。建议每条 JSON 后添加换行符，避免多条数据粘包后无法及时解析

### 5. GitHub 链接无法跳转

请检查 `appInfo.repositoryUrl` 是否已经配置为有效的仓库地址。非 H5 和非 App 平台可能不会直接打开浏览器，此时页面会将仓库地址复制到剪贴板

## 开源信息

本项目以开源方式维护，默认仓库地址为：

```text
https://github.com/Bifangzi/BleMusicPlayerController
```

如需更换仓库地址，请修改 `appInfo.repositoryUrl`

## 许可协议

本项目基于 [MIT License](https://license/) 开源协议
