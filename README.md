按时
安卓端的吃药 / 护肤提醒工具。用 Kotlin + Jetpack Compose 写的，单 Activity，没有账号，数据只存在本机。

能做什么
录入：写名称、选类型（药品 / 护肤），用一句话描述用法——「早晚各一次」「隔两天一次」「用五天停五天」「十五天用一次」「饭后一次」都能认；认不出来时提示改用下面的选项手动排，不瞎猜。
排期：每天 / 每 N 天 / 用 N 天停 N 天三种节奏，配合早上、中午、晚上、睡前四个时段（时间可改），从指定开始日期算起。
饭前 / 饭后 / 随餐：按三餐时间自动换算，比饭点早 30 分钟或晚 30 分钟；三餐时间在设置里改，改完已存在的提醒会跟着变。
拍处方：拍一张处方笺或从相册选图，用 ML Kit 中文识别读出药名和用法，列出来让你确认后再加进日历；认不准的那条会标红，点「改一下」手动补。
查看：月历按天标出当日次数，颜色区分药品与护肤；点任意一天看当天安排，勾选打卡、跳过、撤销。
提醒：到点用 AlarmManager 精确闹钟推通知，带「已服用 / 稍后 10 分钟 / 跳过」三个动作，点开直达当天。开机、改时间、跨天后自动重排未来 8 天的提醒。
自检：设置页能看到已排多少条、下一条是什么、通知/准时提醒/后台运行是否被系统限制，还能发一条测试通知或「一分钟后响」验证锁屏收不收得到。
更新：填一个更新地址，App 自己比对版本号，只下载改动的那部分（典型几十到几百 KB），合并后交给系统安装。
构建
需要 JDK 17 与 Android SDK（compileSdk 35、build-tools 35 以上）。

bash


# 项目自带的 local.properties 里指向本机 SDK，换机器请改 sdk.dir
./gradlew :app:assembleDebug
./gradlew :app:testDebugUnitTest
Windows 上没有 Gradle 时可以直接用解压好的分发包：

powershell


$env:ANDROID_HOME="C:\Users\<you>\AppData\Local\Android\Sdk"
& "<gradle-8.9>\bin\gradle.bat" -p "<repo>" :app:assembleDebug
产物：app/build/outputs/apk/debug/app-debug.apk（约 57 MB，其中大部分是 ML Kit 的中文识别模型）。

安装到模拟器 / 手机
bash


adb install -r app/build/outputs/apk/debug/app-debug.apk
adb shell am start -n com.anshi.pill/.MainActivity
第一次打开会申请通知权限和相机权限。Android 12 以上如果系统没给「闹钟与提醒」权限，应用会退回不精确的闹钟并单独提示；设置页会显示状态并提供跳转。

增量更新怎么用
发一个版本
改 app/build.gradle.kts 里的 versionCode 和 versionName，构建出新包。
用上一版安装包当基线，生成补丁和清单：
bash


python tools/make_update.py old.apk new.apk \
    --version-code 2 --version-name 1.1 \
    --notes-file tools/notes.txt \
    --out dist
--notes 也可以直接写，但 Windows 命令行里的中文容易变乱码，建议用 --notes-file（UTF-8 文本）。

把 dist/ 整个目录放到任意静态服务上（Nginx、对象存储、GitHub Pages 都行）：
代码块


dist/update.json      清单：版本号、说明、补丁与整包的地址 / 大小 / SHA-256
dist/patch-2.bin      补丁，只含变动的片段
dist/app-2.apk        完整包，补丁对不上时的兜底
本地自测：

bash


python tools/serve.py dist --port 8765
# 模拟器里访问宿主用 10.0.2.2：http://10.0.2.2:8765/
手机这边
设置 → 更新 → 填更新地址（指向放着 update.json 的目录，末尾带 /）→ 检查更新 → 更新。 App 会先下补丁（界面上显示「只下 N KB」），按内容分块合并出新包，校验 SHA-256 通过后交给系统安装器，用户点一下「更新」即可，数据不动。

地址也可以在打包时写死，用户在设置里就不用填了：改 app/build.gradle.kts 里的

kotlin


buildConfigField("String", "UPDATE_BASE_URL", "\"https://cdn.example.com/anshi/\"")
设置页里手填的地址优先于这个默认值。断线会自动重试（三次，支持断点续传），补丁对不上或校验不过会退到整包。

补丁格式
data/DeltaPatch.kt 与 tools/make_update.py 用的是同一套按内容切块（CDC）的规则：

代码块


GEAR(b)  = ((b * 0x9E3779B1) & 0xFFFFFFFF) ^ 高 15 位异或回去
h        = ((h << 1) + GEAR(b)) & 0xFFFFFFFF
块长 ≥ 2048 且 h & 0x1FFF == 0 时切块，最长 16384
两边切出来的边界在相同内容上一致，所以新包里没变的片段可以直接「从旧包第 X 字节复制 Y 字节」，只有真正改动的块才带上原始字节。补丁文件格式：

代码块


"ANSHI01\0"            8 字节魔数
u32 版本 = 1
u32 新包总长度
u32 操作数
逐个操作：
  u8 0 + u32 偏移 + u32 长度     从旧包复制
  u8 1 + u32 长度 + 字节         新内容
实测：1.0 → 1.1（只改了版本号和少量代码）补丁 93 KB，约等于新包的 1.6‰。

代码结构
代码块


app/src/main/java/com/anshi/pill/
├── data/
│   ├── Model.kt            Med / Dose / Slot / Kind / MealAnchor / MealTimes
│   ├── FreqParser.kt       中文用法 → 结构化频率（含饭前饭后、tid/bid/qd）
│   ├── Schedule.kt         某天有没有这次用药、下一次是什么时候、状态判定
│   ├── PrescriptionParser.kt  处方识别文本 → 药名 + 用法
│   ├── AppState.kt         SharedPreferences + JSON 存储，写入后自动重排提醒
│   ├── DeltaPatch.kt       补丁格式与合并
│   └── Updater.kt          版本检查、下载、合并、交给系统安装
├── notify/
│   └── Reminder.kt         通知渠道、闹钟、广播接收器（到点 / 动作 / 跨天 / 开机）
├── ui/
│   ├── Theme.kt            配色与状态文案
│   ├── CalendarScreen.kt   月历 + 当天列表
│   ├── MedsScreen.kt       清单
│   ├── EditScreen.kt       新增 / 编辑
│   ├── ScanScreen.kt       拍处方 / 选图 / 识别结果
│   └── SettingsScreen.kt   权限与提醒状态、三餐时间、更新、关于
├── MainActivity.kt         底部导航与页面切换
└── AnshiApp.kt             启动时重排提醒
单元测试在 app/src/test/java/com/anshi/pill/data/：FreqParserTest（17 例）、MealTimeTest（6 例）、ScheduleTest（6 例）、PrescriptionParserTest（10 例）、DeltaPatchTest（4 例，含补丁往返）、UpdaterTest（4 例），共 47 例。

已知限制
短信、微信这类消息提醒做不到：通知是本地推送，不是「发消息」。需要的话得接第三方推送或短信网关。
国产 ROM 可能杀后台导致闹钟不响，需要在系统设置里允许自启动与后台运行。
「饭前 / 饭后」是相对三餐时间的固定偏移（早 30 分钟 / 晚 30 分钟），不是相对你真正动筷子的时刻。
处方识别依赖拍照质量，手写处方认不准；结果里已经剔掉表头和签名栏，但还是要人工确认，别当成医嘱。
更新用的是自建分发 + 自研补丁，不是 Google Play 的增量包；签名要和已安装的一致，否则系统会拒绝覆盖安装。
