<script>
import { ipcRenderer } from 'electron'
import Plugin from '../mixins/plugin'
import { colorTheme } from '../composables/theme'

export default {
  name: 'Setting',
  mixins: [Plugin],
  data () {
    return {
      key: 'app',
      removeUserConfigLoading: false,
      reloadLoading: false,
      urlBackup: null,
      personalUrlBackup: null,
      maxLogFileSizeStep: 1, // 单位不同，值不同：GB=1，MB=100
      maxLogFileSizeUnit: [
        {
          label: 'GB',
          value: 'GB',
        },
        {
          label: 'MB',
          value: 'MB',
        },
      ],
    }
  },
  methods: {
    ready (config) {
      this.urlBackup = config.app.remoteConfig.url
      this.personalUrlBackup = config.app.remoteConfig.personalUrl
    },
    getEventKey (event) {
      // 忽略以下键
      switch (event.key) {
        case 'Control':
        case 'Alt':
        case 'Shift':
        case 'Meta': // Window键
        case 'Escape':
        case 'Backspace':
        case 'Tab':
        case 'CapsLock':
        case 'NumLock':
        case 'Enter':
        case 'ArrowUp':
        case 'ArrowDown':
        case 'ArrowLeft':
        case 'ArrowRight':
          return ''
      }

      switch (event.code) {
        // F1 ~ F12
        case 'F1': return 'F1'
        case 'F2': return 'F2'
        case 'F3': return 'F3'
        case 'F4': return 'F4'
        case 'F5': return 'F5'
        case 'F6': return 'F6'
        case 'F7': return 'F7'
        case 'F8': return 'F8'
        case 'F9': return 'F9'
        case 'F10': return 'F10'
        case 'F11': return 'F11'
        case 'F12': return 'F12'

        // 0 ~ 9
        case 'Digit0': return '0'
        case 'Digit1': return '1'
        case 'Digit2': return '2'
        case 'Digit3': return '3'
        case 'Digit4': return '4'
        case 'Digit5': return '5'
        case 'Digit6': return '6'
        case 'Digit7': return '7'
        case 'Digit8': return '8'
        case 'Digit9': return '9'

        case 'Backquote': return '`'
        case 'Minus': return '-'
        case 'Equal': return '='
        case 'Space': return 'Space'

        case 'BracketLeft': return '['
        case 'BracketRight': return ']'
        case 'Backslash': return '\\'
        case 'Semicolon': return ';'
        case 'Quote': return '\''
        case 'Comma': return ','
        case 'Period': return '.'
        case 'Slash': return '/'

        case 'Insert': return 'Insert'
        case 'Delete': return 'Delete'
        case 'Home': return 'Home'
        case 'End': return 'End'
        case 'PageUp': return 'PageUp'
        case 'PageDown': return 'PageDown'

        // 小键盘
        case 'Numpad1': return 'Num1'
        case 'Numpad2': return 'Num2'
        case 'Numpad3': return 'Num3'
        case 'Numpad4': return 'Num4'
        case 'Numpad5': return 'Num5'
        case 'Numpad6': return 'Num6'
        case 'Numpad7': return 'Num7'
        case 'Numpad8': return 'Num8'
        case 'Numpad9': return 'Num9'
        case 'Numpad0': return 'Num0'

        // 不支持监听以下几个键，返回空
        case 'NumpadDivide': // return 'Num/'
        case 'NumpadMultiply': // return 'Num*'
        case 'NumpadDecimal': // return 'Num.'
        case 'NumpadSubtract': // return 'Num-'
        case 'NumpadAdd': // return 'Num+'
          return ''
      }

      // 字母
      if (event.code.startsWith('Key') && event.code.length === 4) {
        return event.key.toUpperCase()
      }

      console.error(`未能识别的按键：key=${event.key}, code=${event.code}, keyCode=${event.keyCode}`)
      return ''
    },
    async disableBeforeInputEvent () {
      clearTimeout(window.enableBeforeInputEventTimeout)
      window.config.disableBeforeInputEvent = true
      window.enableBeforeInputEventTimeout = setTimeout(() => {
        window.config.disableBeforeInputEvent = false
      }, 2000)
    },
    shortcutChange () {
      this.config.app.showHideShortcut = '无'
    },
    shortcutKeyUp (event) {
      event.preventDefault()
      this.disableBeforeInputEvent()
    },
    shortcutKeyDown (event) {
      event.preventDefault()
      this.disableBeforeInputEvent()

      // console.info(`code=${event.code}, key=${event.key}, keyCode=${event.keyCode}`)
      if (event.type !== 'keydown') {
        return
      }

      const key = this.getEventKey(event)
      if (!key) {
        this.config.app.showHideShortcut = '无'
        return
      }

      // 判断 Ctrl、Alt、Shift、Window 按钮是否已按下，如果已按下，则拼接键值
      let shortcut = event.ctrlKey ? 'Ctrl + ' : ''
      if (event.altKey) {
        shortcut += 'Alt + '
      }
      if (event.shiftKey) {
        shortcut += 'Shift + '
      }
      if (event.metaKey) {
        shortcut += 'Meta + '
      }

      // 如果以上按钮都没有按下，并且当前键不是F1、F2、F4、F6~F11时，则直接返回（注：F5已经是刷新页面快捷键、F12已经是打开DevTools的快捷键了）
      if (shortcut === '' && !key.match(/^F([1246-9]|1[01])$/g)) {
        this.config.app.showHideShortcut = '无'
        return
      }

      // 拼接键值
      shortcut += key

      if (shortcut === 'Ctrl + F' || shortcut === 'Shift + F3') {
        shortcut = '无' // 如果是其他已被占用快捷键，则设置为 '无'
      }

      this.config.app.showHideShortcut = shortcut
    },
    async applyBefore () {
      if (!this.config.app.showHideShortcut) {
        this.config.app.showHideShortcut = '无'
      }
    },
    async applyAfter () {
      let reloadLazy = 10

      let theme = this.config.app.theme
      if (theme === 'system') {
        theme = window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'
      }
      colorTheme.value = theme

      // 判断远程配置地址是否变更过，如果是则重载远程配置并重启服务
      if (this.config.app.remoteConfig.url !== this.urlBackup || this.config.app.remoteConfig.personalUrl !== this.personalUrlBackup) {
        await this.$api.config.downloadRemoteConfig()
        await this.reloadConfigAndRestart()
        reloadLazy = 300
        setTimeout(() => window.location.reload(), reloadLazy)
      }

      // 变更 “打开窗口快捷键”
      ipcRenderer.send('change-showHideShortcut', this.config.app.showHideShortcut)
    },
    async openExternal (url) {
      await this.$api.ipc.openExternal(url)
    },
    onAutoStartChange () {
      this.$api.autoStart.enabled(this.config.app.autoStart.enabled)
      this.saveConfig()
    },
    async onRemoteConfigEnabledChange () {
      await this.saveConfig()
      if (this.config.app.remoteConfig.enabled === true) {
        this.reloadLoading = true
        try {
          this.$message.info('开始下载远程配置')
          await this.$api.config.downloadRemoteConfig()
          this.$message.info('下载远程配置成功，开始重启代理服务和系统代理')
          await this.reloadConfigAndRestart()
        } finally {
          this.reloadLoading = false
        }
      } else {
        this.$message.info('远程配置已关闭，开始重启代理服务和系统代理')
        await this.reloadConfigAndRestart()
      }
    },
    async reloadRemoteConfig () {
      if (this.config.app.remoteConfig.enabled === false) {
        return
      }

      this.reloadLoading = true
      try {
        const remoteConfig = {}

        await this.$api.config.readRemoteConfigStr().then((ret) => {
          remoteConfig.old1 = ret
        })
        await this.$api.config.readRemoteConfigStr('_personal').then((ret) => {
          remoteConfig.old2 = ret
        })
        await this.$api.config.downloadRemoteConfig()
        await this.$api.config.readRemoteConfigStr().then((ret) => {
          remoteConfig.new1 = ret
        })
        await this.$api.config.readRemoteConfigStr('_personal').then((ret) => {
          remoteConfig.new2 = ret
        })

        if (remoteConfig.old1 === remoteConfig.new1 && remoteConfig.old2 === remoteConfig.new2) {
          this.$message.info('远程配置没有变化，不做任何处理。')
          this.$message.warn('如果您确实修改了远程配置，请稍等片刻再重试！')
        } else {
          this.$message.success('获取到了最新的远程配置，开始重启代理服务和系统代理')
          await this.reloadConfigAndRestart()
        }
      } finally {
        this.reloadLoading = false
      }
    },
    async restoreFactorySettings () {
      this.$confirm({
        title: '确定要恢复出厂设置吗？',
        width: 610,
        content: h => (
          <div class="restore-factory-settings">
            <hr />
            <p>
              <h3>操作警告：</h3>
              <div>
                该功能将备份您的所有页面的个性化配置，并重载
                <span>默认配置</span>
                及
                <span>远程配置</span>
                ，请谨慎操作！！！
              </div>
            </p>
            <hr />
            <p>
              <h3>找回个性化配置的方法：</h3>
              <div>
                1. 找到备份文件，路径：
                <span>~/.dev-sidecar/config.json.时间戳.bak.json</span>
                <br />
                2. 将该备份文件重命名为
                <span>config.json</span>
                ，再重启软件即可恢复个性化配置。
              </div>
            </p>
          </div>
        ),
        cancelText: '取消',
        okText: '确定',
        onOk: async () => {
          this.removeUserConfigLoading = true
          try {
            const result = await this.$api.config.removeUserConfig()
            if (result) {
              this.config = await this.$api.config.get()
              this.$message.success('恢复出厂设置成功，开始重启代理服务和系统代理')
              await this.reloadConfigAndRestart()
            } else {
              this.$message.info('已是出厂设置，无需恢复')
            }
          } finally {
            this.removeUserConfigLoading = false
          }
        },
        onCancel () {},
      })
    },
    async onMaxLogFileSizeUnitChange (value) {
      if (value === 'MB') {
        this.config.app.maxLogFileSize = Math.ceil((this.config.app.maxLogFileSize || 1024) * 1024) // 转为整数
        this.maxLogFileSizeStep = 100
      } else {
        this.config.app.maxLogFileSize = ((this.config.app.maxLogFileSize || 1024) / 1024).toFixed(2) - 0 // 最多保留2位小数
        this.maxLogFileSizeStep = 1
      }
      this.$refs.maxLogFileSize.focus()
    },
    async onLogFileSavePathSelect () {
      const value = await this.$api.fileSelector.open(this.config.app.logFileSavePath, 'dir')
      if (value != null && value.length > 0) {
        this.config.app.logFileSavePath = value[0]
      }
    },
  },
}
</script>

<template>
  <ds-container>
    <template slot="header">
      设置
    </template>
    <template slot="header-right">
      <a-button class="mr10" icon="profile" @click="openLog()">查看日志</a-button>
    </template>

    <div v-if="config">
      <a-form-item label="开机自启" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-checkbox v-model="config.app.autoStart.enabled" @change="onAutoStartChange">
          本应用开机自启
        </a-checkbox>
      </a-form-item>
      <a-form-item v-if="systemPlatform === 'mac'" label="隐藏Dock图标" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-checkbox v-model="config.app.dock.hideWhenWinClose">
          关闭窗口时隐藏Dock图标(仅限Mac)
        </a-checkbox>
        <div class="form-help">
          修改后需要重启应用
        </div>
      </a-form-item>
      <hr>
      <a-form-item label="远程配置" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-checkbox v-model="config.app.remoteConfig.enabled" @change="onRemoteConfigEnabledChange">
          启用远程配置
        </a-checkbox>
        <div class="form-help">
          应用启动时会向下面的地址请求配置补丁，获得最新的优化后的github访问体验。<br>
          如果您觉得远程配置有安全风险，请关闭此功能，或删除共享远程配置，仅使用个人远程配置。<br>
          配置优先级：本地修改配置  >  个人远程配置  >  共享远程配置 > 默认配置
        </div>
      </a-form-item>
      <a-form-item label="共享远程配置地址" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-input v-model="config.app.remoteConfig.url" :title="config.app.remoteConfig.url" spellcheck="false" />
      </a-form-item>
      <a-form-item label="个人远程配置地址" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-input v-model="config.app.remoteConfig.personalUrl" :title="config.app.remoteConfig.personalUrl" spellcheck="false" />
      </a-form-item>
      <a-form-item label="重载远程配置" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-button :disabled="config.app.remoteConfig.enabled === false" :loading="reloadLoading" icon="sync" @click="reloadRemoteConfig()">
          重载远程配置
        </a-button>
        <div class="form-help">
          注意，部分远程配置文件所在站点，修改内容后可能需要等待一段时间才能生效。<br>
          如果重载远程配置后发现下载的还是修改前的内容，请稍等片刻再重试。
        </div>
      </a-form-item>
      <hr>
      <a-form-item label="主题设置" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-radio-group v-model="config.app.theme" default-value="light" button-style="solid">
          <a-radio-button value="light" title="light">
            亮色
          </a-radio-button>
          <a-radio-button value="dark" title="dark">
            暗色
          </a-radio-button>
          <a-radio-button value="system" title="system">
            跟随系统
          </a-radio-button>
        </a-radio-group>
      </a-form-item>
      <a-form-item label="首页提示" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-radio-group v-model="config.app.showShutdownTip" default-value="true" button-style="solid">
          <a-radio-button :value="true">
            显示
          </a-radio-button>
          <a-radio-button :value="false">
            隐藏
          </a-radio-button>
        </a-radio-group>
        <div class="form-help">
          是否显示首页的警告提示
        </div>
      </a-form-item>
      <a-form-item v-if="!isLinux()" label="关闭策略" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-radio-group v-model="config.app.closeStrategy" default-value="0" button-style="solid">
          <a-radio-button :value="0">
            弹出提示
          </a-radio-button>
          <a-radio-button :value="1">
            直接退出
          </a-radio-button>
          <a-radio-button :value="2">
            最小化到系统托盘
          </a-radio-button>
        </a-radio-group>
        <div class="form-help">
          点击窗口右上角关闭按钮的效果
        </div>
      </a-form-item>
      <hr>
      <a-form-item label="打开窗口快捷键" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-input v-model="config.app.showHideShortcut" spellcheck="false" @change="shortcutChange" @keydown="shortcutKeyDown" @keyup="shortcutKeyUp" />
        <div class="form-help">
          部分快捷键已被占用：<code>F5</code>、<code>F12</code>、<code>Ctrl+F</code>、<code>F3</code>、<code>Shift+F3</code>
        </div>
      </a-form-item>
      <a-form-item label="启动时窗口状态" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-radio-group v-model="config.app.startShowWindow" default-value="true" button-style="solid">
          <a-radio-button :value="true">
            打开窗口
          </a-radio-button>
          <a-radio-button :value="false">
            隐藏窗口
          </a-radio-button>
        </a-radio-group>
        <div class="form-help">
          启动软件时，是否打开窗口。提示：如果设置为隐藏窗口，可点击系统托盘小图标打开窗口。
        </div>
      </a-form-item>
      <a-form-item label="启动时窗口大小" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-input-number v-model="config.app.windowSize.width" :step="50" :min="600" :max="2400" :precision="0" spellcheck="false" />&nbsp;×
        <a-input-number v-model="config.app.windowSize.height" :step="50" :min="500" :max="2000" :precision="0" spellcheck="false" />
      </a-form-item>
      <hr>
      <a-form-item label="自动检查更新" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-radio-group v-model="config.app.autoChecked" default-value="light" button-style="solid">
          <a-radio-button :value="true">
            开启
          </a-radio-button>
          <a-radio-button :value="false">
            关闭
          </a-radio-button>
        </a-radio-group>
        <div class="form-help">
          开启自动检查更新后，每次应用启动时会检查一次更新，如有新版本，则会弹出提示。
        </div>
      </a-form-item>
      <a-form-item label="忽略预发布版本" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-radio-group v-model="config.app.skipPreRelease" default-value="light" button-style="solid">
          <a-radio-button :value="true">
            忽略
          </a-radio-button>
          <a-radio-button :value="false">
            不忽略
          </a-radio-button>
        </a-radio-group>
        <div class="form-help">
          预发布版本号为带有 “<code>-</code>” 的版本。注：该配置只对当前版本为正式版本时有效。
        </div>
      </a-form-item>
      <hr>
      <a-form-item label="日志文件保存目录" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-input-search
          v-model="config.app.logFileSavePath" enter-button="选择"
          :title="config.app.logFileSavePath" spellcheck="false"
          @search="onLogFileSavePathSelect"
        />
        <div class="form-help">
          修改后，重启DS才生效！<br>
          注意：原目录中的文件不会自动转移到新的目录，请自行转移或删除。
        </div>
      </a-form-item>
      <a-form-item label="最大日志文件大小" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-input-number ref="maxLogFileSize" v-model="config.app.maxLogFileSize" :step="maxLogFileSizeStep" :min="0" spellcheck="false" />
        <a-select v-model="config.app.maxLogFileSizeUnit" class="ml5" @change="onMaxLogFileSizeUnitChange">
          <a-select-option v-for="(item) of maxLogFileSizeUnit" :key="item.value" :value="item.value">
            {{ item.label }}
          </a-select-option>
        </a-select>
        <div class="form-help">
          修改后，重启DS才生效！<br>
          单个日志文件的大小限制，达到限制时会执行备份和清理程序；配置为0时，表示不限制大小。
        </div>
      </a-form-item>
      <a-form-item label="保留日志文件数" :label-col="labelCol" :wrapper-col="wrapperCol">
        <a-input-number v-model="config.app.keepLogFileCount" :step="1" :min="0" :precision="0" spellcheck="false" />
        <div class="form-help">
          修改后，重启DS才生效，<code>隔天</code>或<code>达到日志文件大小限制</code>时，才会触发清理程序！
        </div>
      </a-form-item>
    </div>
    <template slot="footer">
      <div class="footer-bar">
        <a-button :loading="removeUserConfigLoading" class="mr10" icon="sync" @click="restoreFactorySettings()">
          恢复出厂设置
        </a-button>
        <a-button :loading="resetDefaultLoading" class="mr10" icon="sync" @click="resetDefault()">
          恢复默认
        </a-button>
        <a-button :loading="applyLoading" icon="check" type="primary" @click="apply()">
          应用
        </a-button>
      </div>
    </template>
  </ds-container>
</template>
