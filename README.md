# 城发协同 · 安卓 App（WebView 壳）

把后台公网地址 `http://106.52.72.40/` 打包成一个安卓 App。本质是 WebView 容器，
**网页端完全独立运行，本 App 只是套壳，不影响网页版任何功能。**

- 桌面名称：**城发协同**
- 包名：`com.chengfaedu.collabsync`
- 最低系统：Android 8.0（API 26）
- 技术栈：原生 WebView + AndroidX，无内嵌业务代码

---

## 一、更换后台地址（重要）

目前暂用公网 IP。后续上域名 / HTTPS，只改一处即可：

`app/src/main/res/values/strings.xml`

```xml
<string name="app_url">http://106.52.72.40/</string>
```

改成例如 `https://collab.example.com/` 后重新构建即可。

> 切到 HTTPS 后，建议把 `app/src/main/res/xml/network_security_config.xml` 里的
> `cleartextTrafficPermitted` 改回 `false`，并同步去掉 `AndroidManifest.xml` 中的
> `android:usesCleartextTraffic="true"`。

---

## 二、三种构建方式（任选其一）

### 方式 A：Android Studio（最省事，推荐）
1. 用 Android Studio 打开本目录（`android-app`）。
2. 等待 Gradle 同步完成（会自动下载 Gradle 8.9 与依赖）。
3. 菜单 `Build → Build Bundle(s) / APK(s) → Build APK(s)`。
4. 构建完成后右下角弹窗 `locate`，拿到 `app-debug.apk`。
5. 手机「设置 → 允许未知来源」后，用数据线 / 微信 / 邮箱把 apk 传过去安装。

### 方式 B：命令行（需本机装好 JDK17 + Android SDK）
```bash
# 1) 配置 SDK 路径（Windows 用 set，macOS/Linux 用 export）
export ANDROID_HOME=/path/to/android-sdk
export ANDROID_SDK_ROOT=/path/to/android-sdk

# 2) 调试包（无需签名，可直接装手机内部测试）
./gradlew assembleDebug
# 产物：app/build/outputs/apk/debug/app-debug.apk

# 3) 正式包（需先按第四节生成签名，见 app/build.gradle 的 signingConfigs）
./gradlew assembleRelease
# 产物：app/build/outputs/apk/release/app-release.apk
```

### 方式 C：GitHub Actions（云端一键出包，本机零安装）
1. 把本目录推送到一个 GitHub 仓库。
2. 仓库 `Actions` 标签页找到 `Build Android APK`，点 `Run workflow`。
3. 跑完后到 `Artifacts` 下载 `chengfa-collab-debug-apk`（里面是 debug apk）。
4. 正式签名版：按 `build-apk.yml` 注释，在仓库
   `Settings → Secrets and variables → Actions` 里配置
   `KEYSTORE_BASE64 / KEY_ALIAS / KEYSTORE_PASSWORD / KEY_PASSWORD`，
   并取消 `release:` job 的注释后重新运行。

---

## 三、调试包 vs 正式包

| 类型 | 是否需要签名 | 能否上架应用商店 | 内部安装 |
|------|--------------|------------------|----------|
| debug（`assembleDebug`） | 系统自动 debug 签名 | 否 | 可直接装，需开未知来源 |
| release（`assembleRelease`） | 需自有 keystore | 是 | 可直接装 |

内部使用建议先发 **debug 包** 给同事测试；正式分发再打 **release 包**。

---

## 四、生成正式签名（release 用）

需要本机有 JDK（keytool 随 JDK 提供）：

```bash
keytool -genkeypair -v \
  -keystore release.keystore \
  -alias chengfa \
  -keyalg RSA -keysize 2048 -validity 10000

# 把 keystore 转成 base64（用于 GitHub Actions _secret）
# Windows(PowerShell):  [Convert]::ToBase64String([IO.File]::ReadAllBytes("release.keystore"))
# macOS/Linux:          base64 -w0 release.keystore
```

然后在 `app/build.gradle` 的 `android { }` 内补一段签名配置（已预留注释位）：

```groovy
signingConfigs {
    release {
        storeFile file("release.keystore")
        storePassword System.getenv("KEYSTORE_PASSWORD")
        keyAlias System.getenv("KEY_ALIAS")
        keyPassword System.getenv("KEY_PASSWORD")
    }
}
buildTypes {
    release {
        signingConfig signingConfigs.release
        // ...
    }
}
```

---

## 五、权限与说明

- `INTERNET`：加载后台地址（必需）。
- `READ/WRITE_EXTERNAL_STORAGE`：仅旧系统下载附件用，Android 10+ 由 DownloadManager 接管。
- `POST_NOTIFICATIONS`：Android 13+ 下载完成提示（可选，拒绝不影响核心功能）。
- **同域**链接在 App 内打开；**外链**自动跳系统浏览器，避免变成通用浏览器。
- 支持网页内文件上传（`<input type=file>`）、附件下载、相机/麦克风授权请求。

---

## 六、目录结构

```
android-app/
├── build.gradle                 # 工程级（AGP 8.7.3）
├── settings.gradle
├── gradle.properties
├── gradlew / gradlew.bat         # 一键构建脚本（已带 wrapper）
├── gradle/wrapper/               # Gradle 8.9 wrapper
├── .github/workflows/build-apk.yml
└── app/
    ├── build.gradle
    └── src/main/
        ├── AndroidManifest.xml
        ├── java/com/chengfaedu/collabsync/MainActivity.java
        ├── res/values/{strings,colors,themes}.xml
        ├── res/layout/activity_main.xml
        ├── res/drawable/ic_launcher_*.xml
        ├── res/mipmap-anydpi-v26/ic_launcher*.xml
        └── res/xml/network_security_config.xml
```
