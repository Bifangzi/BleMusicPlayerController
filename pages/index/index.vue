<template>
	<view class="content">
		<view class="status-card">
			<text class="title">BLE 蓝牙控制</text>
			<text class="status">状态：{{ statusText }}</text>
			<text v-if="deviceName || deviceId" class="device">已选设备：{{ deviceName || '未命名设备' }}</text>
			<text v-if="deviceId" class="device-id">{{ deviceId }}</text>
			<text v-if="listeningTargetText" class="device-id">当前监听：{{ listeningTargetText }}</text>
			<text v-if="writeTargetText" class="device-id">当前写入：{{ writeTargetText }}</text>
		</view>

		<view class="music-player-card">
			<view class="player-header">
				<view class="disc-box">
					<image
						v-if="playerAssets.cover"
						class="disc-image"
						:src="playerAssets.cover"
						mode="aspectFill"
					/>
					<text v-else class="disc-symbol">♪</text>
				</view>

				<view class="player-info">
					<text class="player-label">当前播放</text>
					<text class="player-name">{{ player.name || '暂无播放音乐' }}</text>
					<view class="player-meta-row">
						<text :class="['player-state', isPlaying ? 'player-state-playing' : 'player-state-paused']">
							{{ isPlaying ? '播放中' : '已暂停' }}
						</text>
						<text class="player-volume">音量：{{ player.volume }}</text>
					</view>
				</view>
			</view>

			<view class="player-progress-row">
				<text class="player-time">{{ player.play_time || FormatPlayerTime(player.play_ms) }}</text>
				<view class="player-progress-box">
					<progress
						class="player-progress"
						:percent="safeProgress"
						stroke-width="6"
						border-radius="10"
						active
						active-mode="forwards"
					/>
				</view>
				<text class="player-time">{{ player.total_time || FormatPlayerTime(player.total_ms) }}</text>
			</view>

			<view class="player-control-row">
				<view
					v-for="btn in playerButtonList"
					:key="btn.key"
					:class="['player-control-btn', btn.className, GetPlayerButtonStateClass(btn)]"
					@click="SendPlayerCommand(btn.key)"
				>
					<image
						v-if="GetPlayerButtonIcon(btn)"
						class="player-control-image"
						:src="GetPlayerButtonIcon(btn)"
						mode="aspectFit"
					/>
					<text v-else class="player-control-text">{{ GetPlayerButtonText(btn) }}</text>
				</view>
			</view>
		</view>

		<button type="primary" @click="InitBluetooth">连接设备</button>
		<!-- <button @click="OpenDevicePopup">2 搜索并连接蓝牙设备</button>
		<button :disabled="!connected" @click="GetServices">3 获取蓝牙服务</button>
		<button :disabled="!connected || !serviceId" @click="GetCharacteristics">4 获取特征值</button> -->

		<view class="select-box">
			<text class="section-title">服务与监听对象选择</text>

			<view class="selector-row">
				<text class="selector-label">服务</text>
				<picker mode="selector" :range="servicePickerRange" :value="selectedServiceIndex" @change="OnServicePickerChange">
					<view class="picker-value">{{ currentServiceText }}</view>
				</picker>
			</view>

			<view class="selector-row">
				<text class="selector-label">写入对象</text>
				<picker mode="selector" :range="writePickerRange" :value="selectedWriteIndex" @change="OnWritePickerChange">
					<view class="picker-value">{{ currentWriteText }}</view>
				</picker>
			</view>

			<view class="selector-row">
				<text class="selector-label">监听对象</text>
				<picker mode="selector" :range="notifyPickerRange" :value="selectedNotifyIndex" @change="OnNotifyPickerChange">
					<view class="picker-value">{{ currentNotifyText }}</view>
				</picker>
			</view>

			<view class="select-actions">
				<button :disabled="!connected || !notifyCharacteristicId" type="primary" @click="Notify">监听当前对象</button>
				<button :disabled="!activeNotifyCharacteristicId" type="warn" @click="StopNotify">停止监听</button>
			</view>
		</view>

		<!-- <view class="command-box">
			<text class="section-title">五个自定义命令按钮</text>
			<view class="command-row" v-for="(item, index) in commandList" :key="index">
				<input class="command-input" v-model="item.text" :placeholder="'请输入命令' + (index + 1)" />
				<button class="command-button" type="primary" :disabled="!canSend" @click="SendQuickCommand(item, index)">
					发送{{ index + 1 }}
				</button>
			</view>
		</view>

		<view class="send-box">
			<input class="send-input" v-model="sendText" placeholder="请输入临时发送内容" />
			<button type="primary" :disabled="!canSend" @click="SendText">发送输入内容</button>
		</view> -->

		<button :disabled="!connected" type="warn" @click="CloseBle">断开连接</button>

		<!-- <view class="msg-box">
			<text class="msg-title">接收内容</text>
			<text class="msg-text">字符串：{{ message || '暂无' }}</text>
			<text class="msg-text">HEX：{{ messageHex || '暂无' }}</text>
		</view> -->

		<view class="footer-info-card">
			<view class="footer-main">
				<image
					v-if="developerInfo.avatar"
					class="footer-avatar"
					:src="developerInfo.avatar"
					mode="aspectFill"
				/>
				<view v-else class="footer-avatar-placeholder">
					<text class="footer-avatar-text">{{ developerInitial }}</text>
				</view>

				<view class="footer-profile">
					<text class="footer-name">{{ developerInfo.name }}</text>
					<text class="footer-role">{{ developerInfo.role }}</text>
					<text class="footer-desc">{{ developerInfo.description }}</text>
				</view>
			</view>

			<view class="footer-meta-grid">
				<view class="footer-meta-item">
					<text class="footer-meta-label">版本号</text>
					<text class="footer-meta-value">{{ appInfo.version }}</text>
				</view>
				<view class="footer-meta-item">
					<text class="footer-meta-label">编译时间</text>
					<text class="footer-meta-value">{{ appInfo.buildTime }}</text>
				</view>
				<view class="footer-meta-item">
					<text class="footer-meta-label">硬件平台</text>
					<text class="footer-meta-value">{{ appInfo.firmware }}</text>
				</view>
				<view class="footer-meta-item">
					<text class="footer-meta-label">通信协议</text>
					<text class="footer-meta-value">{{ appInfo.protocol }}</text>
				</view>
			</view>

			<!-- <text class="footer-copyright">{{ appInfo.copyright }}</text> -->
			
			<view class="footer-opensource" @click="OpenSourceRepository">
							<image
								class="footer-github-icon"
								:src="appInfo.githubIcon"
								mode="aspectFit"
							/>
							<view class="footer-opensource-text">
								<text class="footer-opensource-title">{{ appInfo.openSourceTitle }}</text>
								<text class="footer-opensource-desc">{{ appInfo.openSourceDesc }}</text>
							</view>
							<text class="footer-opensource-arrow">›</text>
						</view>
			
		</view>

		<uni-popup ref="popup" type="bottom" background-color="#FFF">
			<view class="popup-content">
				<view class="popup-header">
					<text class="popup-title">附近 BLE 设备</text>
					<text class="popup-subtitle">点击设备后会自动连接，并读取服务和特征值</text>
				</view>

				<view class="popup-actions">
					<button type="primary" size="mini" :disabled="connected" @click="StartSearchBluetooth">开始搜索</button>
					<button type="warn" size="mini" @click="StopSearchBluetooth">停止搜索</button>
				</view>

				<scroll-view class="box" scroll-y show-scrollbar="true">
					<view class="item" v-for="ble in BlueDeviceList" :key="ble.deviceId" @click="ConnectBle(ble)">
						<text class="ble-name">{{ ble.name || ble.localName || '未命名设备' }}</text>
						<text class="ble-info">DeviceID：{{ ble.deviceId }}</text>
						<text class="ble-info">RSSI：{{ ble.RSSI }}</text>
					</view>
					<view v-if="BlueDeviceList.length === 0" class="empty">
						<text>{{ searching ? '正在搜索，请稍候...' : '暂无设备，请点击开始搜索' }}</text>
					</view>
				</scroll-view>
			</view>
		</uni-popup>
	</view>
</template>

<script>
export default {
	data() {
		return {
			BlueDeviceList: [],
			searching: false,
			connected: false,
			statusText: '未初始化',

			deviceId: '',
			deviceName: '',
			serviceId: '',
			writeCharacteristicId: '',
			notifyCharacteristicId: '',
			readCharacteristicId: '',

			serviceList: [],
			characteristicList: [],
			writeCharacteristicList: [],
			notifyCharacteristicList: [],
			readCharacteristicList: [],
			selectedServiceIndex: 0,
			selectedWriteIndex: 0,
			selectedNotifyIndex: 0,
			activeNotifyServiceId: '',
			activeNotifyCharacteristicId: '',

			// ESP32 常用 Nordic UART Service UUID：RX 用于手机写入，TX 用于设备 notify 返回
			// 如果你的模块使用其他 UUID，可以只修改下面三个默认值，页面仍然支持手动选择服务/特征值。
			defaultServiceId: '6E400001-B5A3-F393-E0A9-E50E24DCCA9E',
			defaultWriteCharacteristicId: '6E400002-B5A3-F393-E0A9-E50E24DCCA9E',
			defaultNotifyCharacteristicId: '6E400003-B5A3-F393-E0A9-E50E24DCCA9E',

			// 调试区仍保留五个命令输入框，默认值与下方播放器控制按钮一致。
			commandList: [
				{ text: 'vol+down' },
				{ text: 'music+up' },
				{ text: 'music+play' },
				{ text: 'music+down' },
				{ text: 'vol+up' }
			],

			// ===================== 播放器控制命令接口 =====================
			// 后续如果 P4 端命令协议变化，只需要改这里，界面按钮会自动使用新命令。
			playerCommandMap: {
				volDown: 'vol+down',
				prev: 'music+up',
				playPause: 'music+play',
				next: 'music+down',
				volUp: 'vol+up'
			},

			// ===================== 播放器按钮样式/图片接口 =====================
			// icon：普通按钮图片；activeIcon/inactiveIcon：播放/暂停按钮在播放/暂停状态下的图片。
			// 图片留空时使用内置文字图标，便于你后续直接替换为 /static/xxx.png。
			playerButtonList: [
				{ key: 'volDown', text: '－', icon: '', className: 'player-btn-volume-down' },
				{ key: 'prev', text: '⏮', icon: '', className: 'player-btn-prev' },
				{
					key: 'playPause',
					text: '▶',
					activeText: 'Ⅱ',
					inactiveText: '▶',
					icon: '',
					activeIcon: '',
					inactiveIcon: '',
					className: 'player-btn-play'
				},
				{ key: 'next', text: '⏭', icon: '', className: 'player-btn-next' },
				{ key: 'volUp', text: '＋', icon: '', className: 'player-btn-volume-up' }
			],

			playerAssets: {
				cover: ''
			},

			// 播放器状态，由 P4 上传的 music_status JSON 自动刷新。
			player: {
				ready: false,
				state: 'paused',
				name: '',
				volume: 0,
				play_ms: 0,
				total_ms: 0,
				play_time: '00:00',
				total_time: '00:00',
				time_text: '00:00/00:00',
				progress: 0,
				index: 0,
				count: 0
			},

			notifyTextBuffer: '',

			// ===================== 底部信息展示接口 =====================
			// avatar 可以配置为 /static/avatar.png 或网络图片；留空时使用名称首字母占位。
			developerInfo: {
				avatar: '/static/avatar-128.jpg',
				name: 'ESP32-P4C6 Music Player',
				// role: 'Embedded System Developer',
				role: 'Developed by BI9BCY',
				description: '基于 ESP32P4 与 ESP32C6 BLE UART 的音乐播放器控制端'
			},
			appInfo: {
				version: 'v2.1.3',
				buildTime: '2026-05-27',
				firmware: 'ESP32-P4C6',
				protocol: 'BLE Notify / UART JSON',
				// copyright: '© 2026 ESP32P4 Music Player'
				
				// 开源信息展示接口：仓库地址、文案和图标均可通过 SetAppInfo 动态修改。
				repositoryUrl: 'https://github.com/zz97303998/ESP32P4-Music-Player',
				openSourceTitle: 'Open Source on GitHub',
				openSourceDesc: '本项目采用开源方式维护，点击查看源码仓库',
				githubIcon: '/static/github_icon.png'
				
			},

			sendText: '',
			message: '',
			messageHex: '',
			_hasValueListener: false
		}
	},
	computed: {
		developerInitial() {
			const name = (this.developerInfo && this.developerInfo.name) ? this.developerInfo.name.trim() : ''
			if (!name) return 'MP'
			const first = name.charAt(0)
			return /[a-zA-Z0-9]/.test(first) ? first.toUpperCase() : first
		},
		canSend() {
			return this.connected && !!this.deviceId && !!this.serviceId && !!this.writeCharacteristicId
		},
		isPlaying() {
			return this.player.state === 'playing'
		},
		safeProgress() {
			let value = Number(this.player.progress || 0)

			if ((!value || value <= 0) && this.player.total_ms > 0) {
				value = (this.player.play_ms * 100) / this.player.total_ms
			}

			if (value < 0) value = 0
			if (value > 100) value = 100
			return value
		},
		servicePickerRange() {
			if (this.serviceList.length === 0) return ['暂无服务，请先获取服务']
			return this.serviceList.map((item, index) => `${index + 1}. ${item.uuid}${item.isPrimary ? '  主服务' : ''}`)
		},
		writePickerRange() {
			if (this.writeCharacteristicList.length === 0) return ['暂无可写特征值']
			return this.writeCharacteristicList.map((item, index) => `${index + 1}. ${item.uuid}  ${this.GetPropertyText(item.properties)}`)
		},
		notifyPickerRange() {
			if (this.notifyCharacteristicList.length === 0) return ['暂无可监听特征值']
			return this.notifyCharacteristicList.map((item, index) => `${index + 1}. ${item.uuid}  ${this.GetPropertyText(item.properties)}`)
		},
		currentServiceText() {
			return this.serviceId || '请选择服务'
		},
		currentWriteText() {
			return this.writeCharacteristicId || '请选择写入对象'
		},
		currentNotifyText() {
			return this.notifyCharacteristicId || '请选择监听对象'
		},
		listeningTargetText() {
			if (!this.activeNotifyCharacteristicId) return ''
			return `${this.activeNotifyServiceId} / ${this.activeNotifyCharacteristicId}`
		},
		writeTargetText() {
			if (!this.writeCharacteristicId) return ''
			return `${this.serviceId} / ${this.writeCharacteristicId}`
		}
	},
	onUnload() {
		this.StopNotify(false)
		this.StopSearchBluetooth()
		this.CloseBle(false)
		uni.closeBluetoothAdapter({})
	},
	methods: {
		OpenDevicePopup() {
			if (this.$refs.popup) this.$refs.popup.open()
		},

		ShowToast(title, icon = 'none') {
			uni.showToast({ title, icon })
		},

		Delay(ms) {
			return new Promise(resolve => setTimeout(resolve, ms))
		},

		NormalizeUUID(uuid) {
			return (uuid || '').toUpperCase()
		},

		IsSameUUID(a, b) {
			return this.NormalizeUUID(a) === this.NormalizeUUID(b)
		},

		GetPropertyText(properties = {}) {
			const arr = []
			if (properties.read) arr.push('read')
			if (properties.write) arr.push('write')
			if (properties.writeNoResponse) arr.push('writeNoResponse')
			if (properties.notify) arr.push('notify')
			if (properties.indicate) arr.push('indicate')
			return arr.join('/')
		},

		InitBluetooth() {
			this.OpenDevicePopup()
			uni.openBluetoothAdapter({
				success: () => {
					this.statusText = '蓝牙初始化成功'
					this.ShowToast('蓝牙初始化成功')
					uni.onBluetoothAdapterStateChange(res => {
						this.searching = res.discovering
						if (!res.available) {
							this.connected = false
							this.statusText = '手机蓝牙不可用或已关闭'
						}
					})
					uni.onBLEConnectionStateChange(res => {
						if (res.deviceId === this.deviceId) {
							this.connected = res.connected
							this.statusText = res.connected ? '设备已连接' : '设备已断开'
							if (res.connected) {
								this.StopSearchBluetooth(false)
							} else {
								this.activeNotifyServiceId = ''
								this.activeNotifyCharacteristicId = ''
							}
						}
					})
				},
				fail: err => {
					console.error('Init Bluetooth Fail', err)
					this.statusText = '蓝牙初始化失败'
					this.ShowToast('请开启手机蓝牙/定位权限', 'error')
				}
			})
		},

		StartSearchBluetooth() {
			if (this.connected) {
				this.StopSearchBluetooth(false)
				this.statusText = '设备已连接，无需继续搜索'
				this.ShowToast('设备已连接，无需继续搜索')
				return
			}

			this.BlueDeviceList = []
			this.searching = true
			this.statusText = '正在搜索 BLE 设备'

			uni.startBluetoothDevicesDiscovery({
				allowDuplicatesKey: false,
				success: () => {
					console.log('Start Search Bluetooth Device')
					uni.onBluetoothDeviceFound(this.DeviceFound)
				},
				fail: err => {
					console.error('Search Bluetooth Device Fail', err)
					this.searching = false
					this.statusText = '搜索失败'
					this.ShowToast('搜索失败，请先初始化蓝牙', 'error')
				}
			})
		},

		DeviceFound(res) {
			const devices = res.devices || []
			devices.forEach(device => {
				if (!device || !device.deviceId) return

				const existIndex = this.BlueDeviceList.findIndex(item => item.deviceId === device.deviceId)
				if (existIndex >= 0) {
					const nextItem = { ...this.BlueDeviceList[existIndex], ...device }
					this.$set ? this.$set(this.BlueDeviceList, existIndex, nextItem) : (this.BlueDeviceList[existIndex] = nextItem)
				} else {
					this.BlueDeviceList.push(device)
				}
			})
		},

		StopSearchBluetooth(updateStatus = true) {
			if (typeof uni.offBluetoothDeviceFound === 'function') {
				uni.offBluetoothDeviceFound(this.DeviceFound)
			}

			if (!this.searching) {
				if (updateStatus && this.connected) this.statusText = '设备已连接'
				return
			}

			uni.stopBluetoothDevicesDiscovery({
				success: () => {
					this.searching = false
					if (updateStatus) {
						this.statusText = this.connected ? '设备已连接' : '已停止搜索'
					}
				},
				fail: err => {
					console.error('Stop Search Bluetooth Fail', err)
					this.searching = false
				}
			})
		},

		async ConnectBle(ble) {
			if (!ble || !ble.deviceId) {
				this.ShowToast('无效设备', 'error')
				return
			}

			this.StopSearchBluetooth()
			this.ResetBleTarget(false)
			this.deviceId = ble.deviceId
			this.deviceName = ble.name || ble.localName || '未命名设备'
			this.statusText = '正在连接设备'

			uni.createBLEConnection({
				deviceId: this.deviceId,
				timeout: 10000,
				success: async () => {
					this.connected = true
					this.StopSearchBluetooth(false)
					this.statusText = '连接成功，正在获取服务'
					this.ShowToast('连接成功')
					if (this.$refs.popup) this.$refs.popup.close()

					try {
						await this.Delay(800)
						await this.GetServices()
						await this.Delay(300)
						await this.GetCharacteristics()
						this.statusText = '设备就绪，请选择监听对象后开启监听'
					} catch (e) {
						console.error('BLE prepare failed', e)
						this.statusText = '连接成功，但服务/特征值获取失败'
						this.ShowToast('请手动选择服务或检查 UUID', 'error')
					}
				},
				fail: err => {
					console.error('Connect Fail', err)
					this.connected = false
					this.statusText = '连接失败'
					this.ShowToast('连接失败', 'error')
				}
			})
		},

		ResetBleTarget(clearDevice = true) {
			if (clearDevice) {
				this.deviceId = ''
				this.deviceName = ''
			}
			this.serviceId = ''
			this.writeCharacteristicId = ''
			this.notifyCharacteristicId = ''
			this.readCharacteristicId = ''
			this.serviceList = []
			this.characteristicList = []
			this.writeCharacteristicList = []
			this.notifyCharacteristicList = []
			this.readCharacteristicList = []
			this.selectedServiceIndex = 0
			this.selectedWriteIndex = 0
			this.selectedNotifyIndex = 0
			this.activeNotifyServiceId = ''
			this.activeNotifyCharacteristicId = ''
		},

		GetServices() {
			return new Promise((resolve, reject) => {
				if (!this.deviceId) {
					this.ShowToast('请先连接设备', 'error')
					reject(new Error('deviceId is empty'))
					return
				}

				uni.getBLEDeviceServices({
					deviceId: this.deviceId,
					success: res => {
						const services = res.services || []
						console.log('device services:', services)
						this.serviceList = services

						const targetService = services.find(item => this.IsSameUUID(item.uuid, this.defaultServiceId)) || services.find(item => item.isPrimary) || services[0]
						if (!targetService) {
							reject(new Error('no service found'))
							return
						}

						this.selectedServiceIndex = services.findIndex(item => this.IsSameUUID(item.uuid, targetService.uuid))
						if (this.selectedServiceIndex < 0) this.selectedServiceIndex = 0
						this.serviceId = targetService.uuid
						this.ClearCharacteristics()
						this.statusText = '已获取服务，请确认或切换服务'
						resolve(services)
					},
					fail: err => {
						console.error('Get Services Fail', err)
						this.ShowToast('获取服务失败', 'error')
						reject(err)
					}
				})
			})
		},

		ClearCharacteristics() {
			this.characteristicList = []
			this.writeCharacteristicList = []
			this.notifyCharacteristicList = []
			this.readCharacteristicList = []
			this.writeCharacteristicId = ''
			this.notifyCharacteristicId = ''
			this.readCharacteristicId = ''
			this.selectedWriteIndex = 0
			this.selectedNotifyIndex = 0
		},

		OnServicePickerChange(e) {
			const index = Number(e.detail.value)
			if (!this.serviceList[index]) return
			this.selectedServiceIndex = index
			this.serviceId = this.serviceList[index].uuid
			this.ClearCharacteristics()
			this.statusText = '已切换服务，正在获取该服务的特征值'
			this.GetCharacteristics()
		},

		GetCharacteristics() {
			return new Promise((resolve, reject) => {
				if (!this.deviceId || !this.serviceId) {
					this.ShowToast('请先获取服务', 'error')
					reject(new Error('deviceId/serviceId is empty'))
					return
				}

				uni.getBLEDeviceCharacteristics({
					deviceId: this.deviceId,
					serviceId: this.serviceId,
					success: res => {
						const characteristics = res.characteristics || []
						console.log('device characteristics:', characteristics)
						this.characteristicList = characteristics
						this.writeCharacteristicList = characteristics.filter(item => item.properties && (item.properties.write || item.properties.writeNoResponse))
						this.notifyCharacteristicList = characteristics.filter(item => item.properties && (item.properties.notify || item.properties.indicate))
						this.readCharacteristicList = characteristics.filter(item => item.properties && item.properties.read)

						const writeChar = this.writeCharacteristicList.find(item => this.IsSameUUID(item.uuid, this.defaultWriteCharacteristicId)) || this.writeCharacteristicList[0]
						const notifyChar = this.notifyCharacteristicList.find(item => this.IsSameUUID(item.uuid, this.defaultNotifyCharacteristicId)) || this.notifyCharacteristicList[0]
						const readChar = this.readCharacteristicList[0]

						this.writeCharacteristicId = writeChar ? writeChar.uuid : ''
						this.notifyCharacteristicId = notifyChar ? notifyChar.uuid : ''
						this.readCharacteristicId = readChar ? readChar.uuid : ''
						this.selectedWriteIndex = writeChar ? this.writeCharacteristicList.findIndex(item => this.IsSameUUID(item.uuid, writeChar.uuid)) : 0
						this.selectedNotifyIndex = notifyChar ? this.notifyCharacteristicList.findIndex(item => this.IsSameUUID(item.uuid, notifyChar.uuid)) : 0
						if (this.selectedWriteIndex < 0) this.selectedWriteIndex = 0
						if (this.selectedNotifyIndex < 0) this.selectedNotifyIndex = 0

						if (!this.writeCharacteristicId) {
							this.statusText = '未找到可写特征值，请切换服务'
							reject(new Error('no writable characteristic found'))
							return
						}

						this.statusText = this.notifyCharacteristicId ? '已获取特征值，请选择监听对象并开启监听' : '已获取写入对象，但没有找到可监听特征值'
						resolve(characteristics)
					},
					fail: err => {
						console.error('Get Characteristics Fail', err)
						this.ShowToast('获取特征值失败', 'error')
						reject(err)
					}
				})
			})
		},

		OnWritePickerChange(e) {
			const index = Number(e.detail.value)
			if (!this.writeCharacteristicList[index]) return
			this.selectedWriteIndex = index
			this.writeCharacteristicId = this.writeCharacteristicList[index].uuid
			this.statusText = '已切换写入对象'
		},

		OnNotifyPickerChange(e) {
			const index = Number(e.detail.value)
			if (!this.notifyCharacteristicList[index]) return
			this.selectedNotifyIndex = index
			this.notifyCharacteristicId = this.notifyCharacteristicList[index].uuid
			this.statusText = '已选择监听对象，请点击开启当前监听对象'
		},

		async Notify() {
			if (!this.deviceId || !this.serviceId || !this.notifyCharacteristicId) {
				this.ShowToast('当前服务没有可监听特征值', 'error')
				return Promise.reject(new Error('notify characteristic is empty'))
			}

			try {
				if (this.activeNotifyCharacteristicId && (!this.IsSameUUID(this.activeNotifyServiceId, this.serviceId) || !this.IsSameUUID(this.activeNotifyCharacteristicId, this.notifyCharacteristicId))) {
					await this.StopNotify(false)
				}
			} catch (e) {
				console.warn('Stop previous notify failed', e)
			}

			if (!this._hasValueListener) {
				uni.onBLECharacteristicValueChange(this.OnBLEValueChange)
				this._hasValueListener = true
			}

			return new Promise((resolve, reject) => {
				uni.notifyBLECharacteristicValueChange({
					state: true,
					deviceId: this.deviceId,
					serviceId: this.serviceId,
					characteristicId: this.notifyCharacteristicId,
					success: res => {
						console.log('Notify Success', res)
						this.activeNotifyServiceId = this.serviceId
						this.activeNotifyCharacteristicId = this.notifyCharacteristicId
						this.statusText = '消息监听已开启'
						this.ShowToast('监听已开启')
						resolve(res)
					},
					fail: err => {
						console.error('Notify Fail', err)
						this.ShowToast('监听失败，请换一个监听对象', 'error')
						reject(err)
					}
				})
			})
		},

		StopNotify(showToast = true) {
			return new Promise(resolve => {
				if (!this.deviceId || !this.activeNotifyServiceId || !this.activeNotifyCharacteristicId) {
					resolve()
					return
				}

				uni.notifyBLECharacteristicValueChange({
					state: false,
					deviceId: this.deviceId,
					serviceId: this.activeNotifyServiceId,
					characteristicId: this.activeNotifyCharacteristicId,
					success: res => {
						console.log('Stop Notify Success', res)
						this.activeNotifyServiceId = ''
						this.activeNotifyCharacteristicId = ''
						this.statusText = '已停止监听'
						if (showToast) this.ShowToast('已停止监听')
						resolve(res)
					},
					fail: err => {
						console.error('Stop Notify Fail', err)
						this.activeNotifyServiceId = ''
						this.activeNotifyCharacteristicId = ''
						resolve(err)
					}
				})
			})
		},

		OnBLEValueChange(res) {
			console.log('BLE value changed:', res)
			if (this.deviceId && res.deviceId && res.deviceId !== this.deviceId) return
			if (this.activeNotifyServiceId && res.serviceId && !this.IsSameUUID(res.serviceId, this.activeNotifyServiceId)) return
			if (this.activeNotifyCharacteristicId && res.characteristicId && !this.IsSameUUID(res.characteristicId, this.activeNotifyCharacteristicId)) return

			this.messageHex = this.ArrayBufferToHex(res.value)
			this.message = this.ArrayBufferToString(res.value)
			this.HandleBleNotifyText(this.message)
		},

		HandleBleNotifyText(text) {
			if (!text) return

			this.notifyTextBuffer += text
			const lines = this.notifyTextBuffer.split('\n')
			this.notifyTextBuffer = lines.pop() || ''

			lines.forEach(line => {
				const item = line.trim()
				if (!item) return

				try {
					const data = JSON.parse(item)
					this.HandleMusicStatus(data)
				} catch (e) {
					// 保留非 JSON 数据在“接收内容”区域显示，不影响普通调试信息。
					console.log('收到非 JSON 数据或 JSON 尚未完整:', item)
				}
			})
		},

		HandleMusicStatus(data) {
			if (!data || data.type !== 'music_status') return

			this.player.ready = data.ready === true
			this.player.state = data.state || 'paused'
			this.player.name = data.name || ''
			this.player.volume = Number(data.volume || 0)
			this.player.play_ms = Number(data.play_ms || 0)
			this.player.total_ms = Number(data.total_ms || 0)
			this.player.play_time = data.play_time || this.FormatPlayerTime(this.player.play_ms)
			this.player.total_time = data.total_time || this.FormatPlayerTime(this.player.total_ms)
			this.player.time_text = data.time_text || `${this.player.play_time}/${this.player.total_time}`
			this.player.progress = Number(data.progress || 0)
			this.player.index = Number(data.index || 0)
			this.player.count = Number(data.count || 0)
		},

		SendPlayerCommand(key) {
			const command = this.playerCommandMap[key]
			if (!command) {
				this.ShowToast('未配置该按钮命令', 'error')
				return
			}
			this.Send(command, '播放器命令')
		},

		GetPlayerButtonIcon(btn) {
			if (!btn) return ''

			if (btn.key === 'playPause') {
				if (this.isPlaying && btn.activeIcon) return btn.activeIcon
				if (!this.isPlaying && btn.inactiveIcon) return btn.inactiveIcon
			}

			return btn.icon || ''
		},

		GetPlayerButtonText(btn) {
			if (!btn) return ''

			if (btn.key === 'playPause') {
				return this.isPlaying ? (btn.activeText || 'Ⅱ') : (btn.inactiveText || '▶')
			}

			return btn.text || ''
		},

		GetPlayerButtonStateClass(btn) {
			if (!btn || btn.key !== 'playPause') return ''
			return this.isPlaying ? 'player-btn-play-active' : 'player-btn-play-paused'
		},

		FormatPlayerTime(ms) {
			let totalSec = Math.floor(Number(ms || 0) / 1000)
			if (totalSec < 0) totalSec = 0

			const min = Math.floor(totalSec / 60)
			const sec = totalSec % 60

			return `${String(min).padStart(2, '0')}:${String(sec).padStart(2, '0')}`
		},

		// 对外暴露：修改播放器按钮命令，不需要改发送函数。
		SetPlayerCommandMap(commandMap) {
			this.playerCommandMap = {
				...this.playerCommandMap,
				...(commandMap || {})
			}
		},

		// 对外暴露：修改按钮图片、文字或额外样式类。
		SetPlayerButtonAssets(buttonAssets) {
			const assets = buttonAssets || {}
			this.playerButtonList = this.playerButtonList.map(btn => {
				const custom = assets[btn.key]
				return custom ? { ...btn, ...custom } : btn
			})
		},

		// 对外暴露：统一设置底部开发者与版本信息。
		SetFooterInfo(options = {}) {
			if (options.developerInfo) {
				this.developerInfo = {
					...this.developerInfo,
					...options.developerInfo
				}
			}

			if (options.appInfo) {
				this.appInfo = {
					...this.appInfo,
					...options.appInfo
				}
			}
		},

		// 对外暴露：单独设置开发者信息。
		SetDeveloperInfo(info = {}) {
			this.developerInfo = {
				...this.developerInfo,
				...info
			}
		},

		// 对外暴露：单独设置应用版本、编译时间与协议信息。
		SetAppInfo(info = {}) {
			this.appInfo = {
				...this.appInfo,
				...info
			}
		},
		
		OpenSourceRepository() {
			const url = this.appInfo && this.appInfo.repositoryUrl ? this.appInfo.repositoryUrl.trim() : ''
			if (!url) {
				this.ShowToast('未配置仓库地址', 'error')
				return
			}

			// #ifdef H5
			if (typeof window !== 'undefined' && window.open) {
				window.open(url, '_blank')
				return
			}
			// #endif

			// #ifdef APP-PLUS
			if (typeof plus !== 'undefined' && plus.runtime && plus.runtime.openURL) {
				plus.runtime.openURL(url)
				return
			}
			// #endif

			uni.setClipboardData({
				data: url,
				success: () => this.ShowToast('仓库地址已复制')
			})
		},


		SendText() {
			this.Send(this.sendText)
		},

		SendQuickCommand(item, index) {
			const command = item && item.text ? item.text : ''
			this.Send(command, `命令${index + 1}`)
		},

		async Send(msg, label = '输入内容') {
			if (!this.canSend) {
				this.ShowToast('设备未就绪，不能发送', 'error')
				return
			}
			if (!msg) {
				this.ShowToast('发送内容为空', 'error')
				return
			}

			try {
				const buffer = this.StringToArrayBuffer(msg)
				const maxChunkSize = 20

				for (let offset = 0; offset < buffer.byteLength; offset += maxChunkSize) {
					const chunk = this.SliceArrayBuffer(buffer, offset, Math.min(offset + maxChunkSize, buffer.byteLength))
					await this.WriteChunk(chunk)
					await this.Delay(30)
				}

				this.statusText = `${label}发送成功：${msg}`
				this.ShowToast('发送成功')
			} catch (err) {
				console.error('Send Fail', err)
				this.statusText = '发送失败'
				this.ShowToast('发送失败，请检查写入对象', 'error')
			}
		},

		WriteChunk(buffer) {
			return new Promise((resolve, reject) => {
				uni.writeBLECharacteristicValue({
					deviceId: this.deviceId,
					serviceId: this.serviceId,
					characteristicId: this.writeCharacteristicId,
					value: buffer,
					success: res => {
						console.log('writeBLECharacteristicValue success', res.errMsg)
						resolve(res)
					},
					fail: err => reject(err)
				})
			})
		},

		Read() {
			if (!this.deviceId || !this.serviceId || !this.readCharacteristicId) {
				this.ShowToast('当前设备没有 read 特征值', 'error')
				return
			}

			uni.readBLECharacteristicValue({
				deviceId: this.deviceId,
				serviceId: this.serviceId,
				characteristicId: this.readCharacteristicId,
				success: res => {
					console.log('Read Success', res)
					this.ShowToast('读取指令已发送')
				},
				fail: err => {
					console.error('Read Fail', err)
					this.ShowToast('读取失败', 'error')
				}
			})
		},

		CloseBle(showToast = true) {
			if (!this.deviceId) return
			this.StopNotify(false)
			uni.closeBLEConnection({
				deviceId: this.deviceId,
				success: () => {
					this.connected = false
					this.statusText = '设备已断开'
					this.activeNotifyServiceId = ''
					this.activeNotifyCharacteristicId = ''
					if (showToast) this.ShowToast('已断开连接')
				},
				fail: err => {
					console.error('Close BLE Fail', err)
				}
			})
		},

		StringToArrayBuffer(str) {
			// 将字符串按 UTF-8 编码成 ArrayBuffer；ASCII 指令也兼容
			const encoded = unescape(encodeURIComponent(str))
			const buffer = new ArrayBuffer(encoded.length)
			const dataView = new DataView(buffer)
			for (let i = 0; i < encoded.length; i++) {
				dataView.setUint8(i, encoded.charCodeAt(i))
			}
			return buffer
		},

		ArrayBufferToString(buffer) {
			const bytes = new Uint8Array(buffer)
			let binary = ''
			for (let i = 0; i < bytes.length; i++) {
				binary += String.fromCharCode(bytes[i])
			}
			try {
				return decodeURIComponent(escape(binary))
			} catch (e) {
				return binary
			}
		},

		ArrayBufferToHex(buffer) {
			return Array.prototype.map.call(new Uint8Array(buffer), bit => ('00' + bit.toString(16)).slice(-2)).join(' ')
		},

		SliceArrayBuffer(buffer, start, end) {
			if (buffer.slice) return buffer.slice(start, end)
			return new Uint8Array(buffer).slice(start, end).buffer
		}
	}
}
</script>

<style>
.content {
	min-height: 100vh;
	box-sizing: border-box;
	padding: 24rpx;
	background-color: #f7f8fa;
}

.status-card,
.msg-box,
.send-box,
.command-box,
.select-box,
.footer-info-card {
	box-sizing: border-box;
	width: 100%;
	padding: 24rpx;
	margin-bottom: 20rpx;
	border-radius: 16rpx;
	background-color: #ffffff;
}

.title {
	display: block;
	font-size: 36rpx;
	font-weight: bold;
	margin-bottom: 12rpx;
	color: #222222;
}

.status,
.device,
.device-id,
.msg-text,
.msg-title,
.section-title {
	display: block;
	font-size: 26rpx;
	line-height: 1.7;
	color: #333333;
	word-break: break-all;
}

.section-title,
.msg-title {
	font-weight: bold;
	margin-bottom: 14rpx;
}

.device-id {
	font-size: 22rpx;
	color: #666666;
}

button {
	margin-bottom: 18rpx;
}

.send-input,
.command-input {
	box-sizing: border-box;
	width: 100%;
	height: 78rpx;
	padding: 0 20rpx;
	border: 1px solid #dddddd;
	border-radius: 12rpx;
	background-color: #ffffff;
}

.send-input {
	margin-bottom: 18rpx;
}

.command-row {
	display: flex;
	align-items: center;
	gap: 14rpx;
	margin-bottom: 16rpx;
}

.command-input {
	flex: 1;
	min-width: 0;
}

.command-button {
	width: 190rpx;
	margin-bottom: 0;
}

.selector-row {
	margin-bottom: 18rpx;
}

.selector-label {
	display: block;
	font-size: 24rpx;
	color: #666666;
	margin-bottom: 8rpx;
}

.picker-value {
	box-sizing: border-box;
	width: 100%;
	min-height: 78rpx;
	padding: 18rpx 20rpx;
	border: 1px solid #dddddd;
	border-radius: 12rpx;
	background-color: #fafafa;
	font-size: 22rpx;
	line-height: 1.4;
	color: #333333;
	word-break: break-all;
}

.select-actions {
	display: flex;
	gap: 16rpx;
}

.select-actions button {
	flex: 1;
}


.music-player-card {
	box-sizing: border-box;
	width: 100%;
	padding: 28rpx;
	margin-bottom: 20rpx;
	border-radius: 24rpx;
	background: linear-gradient(135deg, #172033 0%, #0f172a 100%);
	box-shadow: 0 16rpx 42rpx rgba(15, 23, 42, 0.18);
}

.player-header {
	display: flex;
	align-items: center;
	margin-bottom: 24rpx;
}

.disc-box {
	width: 150rpx;
	height: 150rpx;
	border-radius: 28rpx;
	margin-right: 24rpx;
	overflow: hidden;
	background: linear-gradient(145deg, #334155, #020617);
	display: flex;
	align-items: center;
	justify-content: center;
	flex-shrink: 0;
}

.disc-image {
	width: 100%;
	height: 100%;
}

.disc-symbol {
	font-size: 72rpx;
	line-height: 1;
	color: #e2e8f0;
	font-weight: bold;
}

.player-info {
	flex: 1;
	min-width: 0;
}

.player-label {
	display: block;
	font-size: 22rpx;
	color: #94a3b8;
	margin-bottom: 8rpx;
}

.player-name {
	display: block;
	font-size: 34rpx;
	font-weight: bold;
	line-height: 1.35;
	color: #f8fafc;
	word-break: break-all;
}

.player-meta-row {
	display: flex;
	align-items: center;
	flex-wrap: wrap;
	margin-top: 14rpx;
	gap: 12rpx;
}

.player-state,
.player-volume {
	display: inline-block;
	padding: 6rpx 18rpx;
	border-radius: 999rpx;
	font-size: 22rpx;
}

.player-state-playing {
	color: #86efac;
	background-color: rgba(34, 197, 94, 0.18);
}

.player-state-paused {
	color: #cbd5e1;
	background-color: rgba(148, 163, 184, 0.18);
}

.player-volume {
	color: #bfdbfe;
	background-color: rgba(59, 130, 246, 0.18);
}

.player-progress-row {
	display: flex;
	align-items: center;
	margin: 8rpx 0 28rpx;
}

.player-time {
	width: 92rpx;
	text-align: center;
	font-size: 24rpx;
	color: #cbd5e1;
}

.player-progress-box {
	flex: 1;
	margin: 0 14rpx;
}

.player-progress {
	width: 100%;
}

.player-control-row {
	display: flex;
	align-items: center;
	justify-content: space-between;
}

.player-control-btn {
	width: 88rpx;
	height: 88rpx;
	border-radius: 50%;
	background-color: rgba(30, 41, 59, 0.96);
	display: flex;
	align-items: center;
	justify-content: center;
	box-shadow: 0 10rpx 28rpx rgba(0, 0, 0, 0.22);
}

.player-control-btn:active {
	transform: scale(0.94);
	opacity: 0.82;
}

.player-control-image {
	width: 54rpx;
	height: 54rpx;
}

.player-control-text {
	font-size: 36rpx;
	font-weight: bold;
	color: #f8fafc;
}

.player-btn-play {
	width: 112rpx;
	height: 112rpx;
}

.player-btn-play-active {
	background: linear-gradient(145deg, #2563eb, #0ea5e9);
}

.player-btn-play-paused {
	background: linear-gradient(145deg, #f97316, #fb923c);
}


.footer-info-card {
	position: relative;
	overflow: hidden;
	background: linear-gradient(135deg, #ffffff 0%, #eef6ff 100%);
	border: 1px solid rgba(59, 130, 246, 0.12);
	box-shadow: 0 14rpx 36rpx rgba(15, 23, 42, 0.08);
}

.footer-info-card::before {
	content: '';
	position: absolute;
	right: -70rpx;
	top: -70rpx;
	width: 190rpx;
	height: 190rpx;
	border-radius: 50%;
	background: rgba(59, 130, 246, 0.12);
}

.footer-main {
	position: relative;
	display: flex;
	align-items: center;
	margin-bottom: 22rpx;
}

.footer-avatar,
.footer-avatar-placeholder {
	width: 108rpx;
	height: 108rpx;
	border-radius: 28rpx;
	margin-right: 22rpx;
	flex-shrink: 0;
	overflow: hidden;
}

.footer-avatar-placeholder {
	display: flex;
	align-items: center;
	justify-content: center;
	background: linear-gradient(145deg, #2563eb, #0ea5e9);
	box-shadow: 0 10rpx 28rpx rgba(37, 99, 235, 0.24);
}

.footer-avatar-text {
	font-size: 40rpx;
	font-weight: bold;
	color: #ffffff;
}

.footer-profile {
	flex: 1;
	min-width: 0;
}

.footer-name {
	display: block;
	font-size: 32rpx;
	font-weight: bold;
	color: #0f172a;
	margin-bottom: 6rpx;
}

.footer-role {
	display: block;
	font-size: 23rpx;
	color: #2563eb;
	margin-bottom: 8rpx;
}

.footer-desc {
	display: block;
	font-size: 23rpx;
	line-height: 1.5;
	color: #64748b;
}

.footer-meta-grid {
	position: relative;
	display: grid;
	grid-template-columns: repeat(2, 1fr);
	gap: 14rpx;
	margin-bottom: 18rpx;
}

.footer-meta-item {
	box-sizing: border-box;
	min-height: 94rpx;
	padding: 16rpx;
	border-radius: 16rpx;
	background-color: rgba(255, 255, 255, 0.72);
	border: 1px solid rgba(148, 163, 184, 0.18);
}

.footer-meta-label {
	display: block;
	font-size: 21rpx;
	color: #94a3b8;
	margin-bottom: 8rpx;
}

.footer-meta-value {
	display: block;
	font-size: 23rpx;
	line-height: 1.35;
	color: #1e293b;
	word-break: break-all;
}

/* .footer-copyright {
	position: relative;
	display: block;
	padding-top: 12rpx;
	border-top: 1px solid rgba(148, 163, 184, 0.2);
	font-size: 22rpx;
	color: #94a3b8;
	text-align: center;
} */

.footer-opensource {
	position: relative;
	display: flex;
	align-items: center;
	box-sizing: border-box;
	min-height: 82rpx;
	padding: 16rpx 18rpx;
	border-top: 1px solid rgba(148, 163, 184, 0.18);
	border-radius: 16rpx;
	background-color: rgba(255, 255, 255, 0.72);
}

.footer-opensource:active {
	transform: scale(0.985);
	opacity: 0.82;
}

.footer-github-icon {
	width: 42rpx;
	height: 42rpx;
	margin-right: 16rpx;
	flex-shrink: 0;
}

.footer-opensource-text {
	flex: 1;
	min-width: 0;
}

.footer-opensource-title {
	display: block;
	font-size: 24rpx;
	font-weight: bold;
	line-height: 1.35;
	color: #1e293b;
}

.footer-opensource-desc {
	display: block;
	margin-top: 4rpx;
	font-size: 21rpx;
	line-height: 1.45;
	color: #64748b;
}

.footer-opensource-arrow {
	margin-left: 12rpx;
	font-size: 36rpx;
	line-height: 1;
	color: #94a3b8;
}

.popup-content {
	box-sizing: border-box;
	padding: 24rpx;
}

.popup-header {
	margin-bottom: 20rpx;
}

.popup-title {
	display: block;
	font-size: 32rpx;
	font-weight: bold;
	color: #222222;
}

.popup-subtitle {
	display: block;
	font-size: 24rpx;
	color: #666666;
	margin-top: 8rpx;
}

.popup-actions {
	display: flex;
	gap: 16rpx;
	margin-bottom: 20rpx;
}

.popup-actions button {
	flex: 1;
}

.box {
	width: 100%;
	height: 600rpx;
	box-sizing: border-box;
	border: 1px solid #dddddd;
	border-radius: 12rpx;
	background-color: #ffffff;
}

.item {
	box-sizing: border-box;
	padding: 18rpx;
	border-bottom: 1px solid #eeeeee;
}

.ble-name {
	display: block;
	font-size: 28rpx;
	font-weight: bold;
	color: #222222;
	margin-bottom: 6rpx;
}

.ble-info {
	display: block;
	font-size: 22rpx;
	color: #666666;
	line-height: 1.5;
	word-break: break-all;
}

.empty {
	padding: 40rpx 20rpx;
	text-align: center;
	color: #999999;
}
</style>
