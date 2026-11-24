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



第一次打开会申请通知权限和相机权限。Android 12 以上如果系统没给「闹钟与提醒」权限，应用会退回不精确的闹钟并单独提示；设置页会显示状态并提供跳转。


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
