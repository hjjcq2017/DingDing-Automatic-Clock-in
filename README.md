# DingDing-Automatic-Clock-in

<img width="300" src="https://github.com/georgehuan1994/DingDing-Automatic-Clock-in/blob/master/图片/Screenshot_2020-10-29-19-29-35-361_org.autojs.autojs.jpg"/> <img width="300"  src="https://github.com/georgehuan1994/DingDing-Automatic-Clock-in/blob/master/图片/Scrennshot_20201231094431.png"/>

## 简介
基于AutoJs的钉钉自动打卡、远程打卡脚本，可用于蓝牙考勤机

## 功能
- 定时打卡
- 远程打卡
- 邮件回复打卡结果

## 工具
- AutoJs
- Tasker
- 网易邮箱大师

## 原理
在AutoJs脚本中监听本机通知，在Tasker中创建定时任务发出打卡通知，或在另一设备上发送消息到本机，即可触发脚本中的打卡进程，实现定时打卡和远程打卡。

## 脚本
```javascript
/*
 * @Author: George Huan
 * @Date: 2020-08-03 09:30:30
 * @LastEditTime: 2021-03-05 16:58:06
 * @Description: DingDing-Automatic-Clock-in (Run on AutoJs)
 * @URL: https://github.com/georgehuan1994/DingDing-Automatic-Clock-in
 */

const ACCOUNT = "钉钉账号"
const PASSWORD = "钉钉密码"
const EMAILL_ADDRESS = "用于接收打卡结果的邮箱地址"

const BUNDLE_ID_DD = "com.alibaba.android.rimet"
const BUNDLE_ID_XMSF = "com.xiaomi.xmsf"
const BUNDLE_ID_MAIL = "com.netease.mail"
const BUNDLE_ID_TASKER = "net.dinglisch.android.taskerm"

const NAME_OF_EMAILL_APP = "网易邮箱大师"
const NAME_OF_ATTENDANCE_MACHINE = "前台大门" // 考勤机名称

const LOWER_BOUND = 1 * 60 * 1000 // 最小等待时间：1min
const UPPER_BOUND = 5 * 60 * 1000 // 最大等待时间：5min

// 执行时的屏幕亮度（0-255），需要"修改系统设置"权限
const SCREEN_BRIGHTNESS = 20    

// 是否过滤通知
const NOTIFICATIONS_FILTER = false; 

// BundleId白名单
const BUNDLE_ID_WHITE_LIST = [BUNDLE_ID_DD,BUNDLE_ID_XMSF,BUNDLE_ID_MAIL,BUNDLE_ID_TASKER, ]

const WEEK_DAY = ["Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday",]

// 公司的钉钉CorpId，获取方法见 2020-09-24 更新日志。如果只加入了一家公司，可以不填
const CORP_ID = "" 


// =================== ↓↓↓ 主线程：监听通知 ↓↓↓ ====================

var suspend = false
var needWaiting = true
var currentDate = new Date()

// 检查无障碍权限启动
auto.waitFor("normal")

// 创建运行日志
console.setGlobalLogConfig({
    file: "/sdcard/脚本/Archive/" + getCurrentDate() + "-log.txt"
});

// 监听本机通知
events.observeNotification()    
events.on("notification", function(n) {
    notificationHandler(n)
});

toastLog("监听中，请在日志中查看记录的通知及其内容")

// =================== ↑↑↑ 主线程：监听通知 ↑↑↑ =====================



/**
 * @description 处理通知
 */
function notificationHandler(n) {
    
    var bundleId = n.getPackageName()    // 获取通知包名
    var abstract = n.tickerText          // 获取通知摘要
    var text = n.getText()               // 获取通知文本

    // 过滤BundleId白名单之外的应用所发出的通知
    if (!filterNotification(bundleId, abstract, text)) { 
        return;
    }

    // 监听摘要为 "定时打卡" 的通知，不一定要从 Tasker 中发出通知，日历、定时器等App均可实现
    if (abstract == "定时打卡" && !suspend) { 
        needWaiting = true
        threads.shutDownAll()
        threads.start(function(){
            doClock()
        })
        return;
    }
    
    // 监听文本为 "打卡" 的通知
    if ((bundleId == BUNDLE_ID_MAIL || bundleId == BUNDLE_ID_XMSF) && text == "打卡") { 
        needWaiting = false
        threads.shutDownAll()
        threads.start(function(){
            doClock()
        })
        return;
    }
    
    // 监听文本为 "考勤结果" 的通知 
    if ((bundleId == BUNDLE_ID_MAIL || bundleId == BUNDLE_ID_XMSF) && (text == "Re: 考勤结果" || text == "考勤结果")) {
        threads.shutDownAll()
        threads.start(function(){
            sendEmail("考勤结果", getStorageData("dingding", "clockResult"))
        })
        return;
    }

    // 监听文本为 "暂停" 的通知 
    if ((bundleId == BUNDLE_ID_MAIL || bundleId == BUNDLE_ID_XMSF) && text == "暂停") {
        suspend = true
        console.warn("暂停定时打卡")
        threads.shutDownAll()
        threads.start(function(){
            sendEmail("操作成功", "已暂停定时打卡功能")
        })
        return;
    }

    // 监听文本为 "恢复" 的通知 
    if ((bundleId == BUNDLE_ID_MAIL || bundleId == BUNDLE_ID_XMSF) && text == "恢复") {
        suspend = false
        console.warn("恢复定时打卡")
        threads.shutDownAll()
        threads.start(function(){
            sendEmail("操作成功", "已恢复定时打卡功能")
        })
        return;
    }

    if (text == null) {
        return;
    }
    
    // 监听钉钉返回的考勤结果
    if (bundleId == BUNDLE_ID_DD && text.indexOf("考勤打卡") >= 0) { 
        setStorageData("dingding", "clockResult", text)
        threads.shutDownAll()
        threads.start(function(){
            sendEmail("考勤结果", text)
        })
        return;
    }
}


/**
 * @description 打卡主程序 
 */
function doClock() {

    currentDate = new Date()
    console.log("本地时间：" + getCurrentDate() + " " + getCurrentTime())
    console.log("开始打卡流程！")

    brightScreen()      // 唤醒屏幕
    unlockScreen()      // 解锁屏幕
    stopApp()           // 结束钉钉
    holdOn()            // 随机等待
    signIn()            // 自动登录
    handleUpdata()      // 处理更新
    handleLate()        // 处理迟到
    attendKaoqin()      // 考勤打卡

    if (currentDate.getHours() <= 12) 
    clockIn()           // 上班打卡
    else 
    clockOut()          // 下班打卡
    
    lockScreen()        // 关闭屏幕
}


/**
 * @description 发邮件主程序 
 * @param {*} title 邮件主题
 * @param {*} message 邮件正文
 */
function sendEmail(title, message) {

    console.log("开始发送邮件流程！")

    brightScreen()  // 唤醒屏幕
    unlockScreen()  // 解锁屏幕

    app.sendEmail({
        email: [EMAILL_ADDRESS],
        subject: title,
        text: message
    })
    
    // 等待选择应用界面弹窗出现，如果设置了默认应用就注释掉
    waitForActivity("com.android.internal.app.ChooserActivity")
    
    if (null != textMatches(NAME_OF_EMAILL_APP).findOne(3000)) {
        btn_email = textMatches(NAME_OF_EMAILL_APP).findOnce().parent()
        btn_email.click()
    }
    else {
        console.error("没有找到" + NAME_OF_EMAILL_APP)
        lockScreen()
        return;
    }

    waitForActivity("com.netease.mobimail.activity.MailComposeActivity")
    id("send").findOne().click()

    console.log("正在发送邮件...")
    
    home()
    sleep(1000)
    lockScreen()    // 关闭屏幕
}


/**
 * @description 唤醒设备
 */
function brightScreen() {

    console.log("唤醒设备")
    
    device.setBrightnessMode(0) // 手动亮度模式
    device.setBrightness(SCREEN_BRIGHTNESS)
    device.wakeUpIfNeeded() // 唤醒设备
    device.keepScreenOn()   // 保持亮屏

    console.info("设备已唤醒")
    
    sleep(1000) // 等待屏幕亮起
    if (!device.isScreenOn()) {
        console.warn("设备未唤醒，重试")
        device.wakeUpIfNeeded()
        brightScreen()
    }
    sleep(1000)
}


/**
 * @description 解锁屏幕
 */
function unlockScreen() {

    console.log("解锁屏幕")
    
    gesture(320,[540,device.height * 0.9],[540,device.height * 0.1]) // 上滑解锁
    sleep(1000) // 等待解锁动画完成
    home()
    sleep(1000) // 等待返回动画完成
    
    console.info("屏幕已解锁")
}


/**
 * @description 结束钉钉进程
 */
function stopApp() {

    console.log("结束钉钉进程")

    // Root
    // shell('am force-stop ' + BUNDLE_ID_DD, true) 

    // No Root
    app.openAppSetting(BUNDLE_ID_DD)
    let btn_finish = textMatches(/(.*结束.*)|(.*停止.*)/).clickable(true).findOne() // 直到找到 "结束运行" 按钮，并点击
    if (btn_finish.enabled()) {
        btn_finish.click()
        
        if (null != textMatches("确定").clickable(true).findOne(1000)) { // 点击弹出的对话框中的 "确定" 按钮
            btn_sure = textMatches("确定").clickable(true).findOnce()
            btn_sure.click() 
        }
        if (null != descMatches("确定").clickable(true).findOne(1000)) {
            btn_sure = descMatches("确定").clickable(true).findOnce()
            btn_sure.click() 
        }
        console.info(app.getAppName(BUNDLE_ID_DD) + "已被关闭")
    } 
    else {
        console.info(app.getAppName(BUNDLE_ID_DD) + "未在运行")
    }
    sleep(1000)
    home()
    sleep(1000)
}


/**
 * @description 随机等待
 */
function holdOn(){

    if (!needWaiting) {
        return;
    }

    var randomTime = random(LOWER_BOUND, UPPER_BOUND)
    toastLog(Math.floor(randomTime / 1000) + "秒后启动" + app.getAppName(BUNDLE_ID_DD) + "...")
    sleep(randomTime)
}


/**
 * @description 启动并登陆钉钉
 */
function signIn() {

    app.launchPackage(BUNDLE_ID_DD)
    console.log("正在启动" + app.getAppName(BUNDLE_ID_DD) + "...")
    
    sleep(10000)    // 等待钉钉启动
    handleUpdata()  // 处理更新弹窗

    if (id("et_pwd_login").exists()) {
        console.info("账号未登录")

        var account = id("et_phone_input").findOne()
        account.setText(ACCOUNT)
        console.log("输入账号")

        var password = id("et_pwd_login").findOne()
        password.setText(PASSWORD)
        console.log("输入密码")
        
        var btn_login = id("btn_next").findOne()
        btn_login.click()
        console.log("正在登陆")
    }
    else {
        if (id("menu_tel").exists()) {
            console.info("账号已登录，当前位于消息页面")
            sleep(1000)
        } 
        else {
            console.warn("未检测到消息页面，重试")
            signIn()
        }
    }
}


/**
 * @description 处理钉钉更新弹窗
 */
function handleUpdata(){

    if (null != textMatches(/(.*暂不更新.*)/).clickable(true).findOne(3000)) {
        btn_dontUpdate = textMatches("暂不更新").clickable(true).findOnce()
        btn_dontUpdate.click()
        sleep(1000)
        console.error("发现更新弹窗！请留意新版本的布局变化！")
    }
}


/**
 * @description 处理迟到打卡
 */
function handleLate(){
   
    if (null != textMatches(/(.*迟到打卡.*)/).clickable(true).findOne(1000)) {
        btn_late = textMatches("迟到打卡").clickable(true).findOnce() 
        btn_late.click()
        console.warn("迟到打卡")
    }
}


/**
 * @description 使用 URL Scheme 进入考勤界面
 */
function attendKaoqin(){

    var url_scheme = "dingtalk://dingtalkclient/page/link?url=https://attend.dingtalk.com/attend/index.html"

    if(CORP_ID != "") {
        url_scheme = url_scheme + "?corpId=" + CORP_ID
    }

    var a = app.intent({
        action: "VIEW",
        data: url_scheme,
        //flags: [Intent.FLAG_ACTIVITY_NEW_TASK]
    });
    
    app.startActivity(a);
    console.log("正在进入考勤界面...")
    sleep(6000)
    
    if (null != textMatches("申请").clickable(true).findOne(3000)) {
        console.info("已进入考勤界面")
        sleep(1000)
    }
}


/**
 * @description 上班打卡 
 */
function clockIn() {

    console.log("上班打卡...")
    
    if (null != textContains("休息").findOne(1000)) {
        console.info("textContains：今日休息")
        home()
        sleep(1000)
        return;
    }
    if (null != descContains("休息").findOne(1000)) {
        console.info("descContains：今日休息")
        home()
        sleep(1000)
        return;
    }

    if (null != textContains("已打卡").findOne(1000)) {
        toastLog("已打卡")
        home()
        sleep(1000)
        return;
    }

    console.log("等待连接到考勤机...")
    sleep(2000)
    
    if (null != textContains("未连接").findOne(1000)) {
        console.error("未连接考勤机，重新进入考勤界面！")
        attendKaoqin()
    }

    textContains(NAME_OF_ATTENDANCE_MACHINE).waitFor()
    console.info("已连接考勤机：" + NAME_OF_ATTENDANCE_MACHINE)
    sleep(1000)

    if (null != textMatches(/(.*上班打卡.*)/).clickable(true).findOne(1000)) {
        // btn_clockin = textMatches("上班打卡").clickable(true).findOnce().parent().parent().click()
        // btn_clockin = textMatches("上班打卡").clickable(true).findOnce().parent().click()
        btn_clockin = textMatches("上班打卡").clickable(true).findOnce();
        btn_clockin.click()
        console.log("按下打卡按钮")
        sleep(1000)
    }

    // 因上班打卡按钮有可能获取不到，故使用打卡按钮坐标作为保险操作
    click(Math.floor(device.width / 2),Math.floor(device.height * 0.560))
    sleep(200)
    click(Math.floor(device.width / 2),Math.floor(device.height * 0.563))
    sleep(200)
    click(Math.floor(device.width / 2),Math.floor(device.height * 0.566))
    console.log("使用坐标按下打卡按钮")
    sleep(1000)

    handleLate() // 处理迟到打卡

    home()
    sleep(1000)
}


/**
 * @description 下班打卡 
 */
function clockOut() {

    console.log("下班打卡...")

    if (null != textContains("休息").findOne(1000)) {
        console.info("textContains：今日休息")
        home()
        sleep(1000)
        return;
    }
    if (null != descContains("休息").findOne(1000)) {
        console.info("descContains：今日休息")
        home()
        sleep(1000)
        return;
    }

    if (null != textContains("更新打卡").findOne(1000)) {
        if (null != textContains("早退").findOne(1000)) {
            toastLog("早退，更新打卡记录")
        }
        else {
            home()
            sleep(1000)
            return;
        }
    }

    console.log("等待连接到考勤机...")
    sleep(2000)
    
    if (null != textContains("未连接").findOne(1000)) {
        console.error("未连接考勤机，重新进入考勤界面！")
        attendKaoqin()
    }

    textContains(NAME_OF_ATTENDANCE_MACHINE).waitFor()
    console.info("已连接考勤机：" + NAME_OF_ATTENDANCE_MACHINE)
    sleep(1000)

    if (null != textMatches(/(.*下班打卡.*)/).clickable(true).findOne(1000)) {
        btn_clockout = textMatches("下班打卡").clickable(true).findOnce()
        btn_clockout.click()
        console.log("按下打卡按钮")
        sleep(1000)
    }

    if (null != textContains("早退打卡").clickable(true).findOne(1000)) {
        className("android.widget.Button").text("早退打卡").clickable(true).findOnce().parent().click()
        console.warn("早退打卡")
    }

    home()
    sleep(1000)
}


/**
 * @description 锁屏
 */
function lockScreen(){

    console.log("关闭屏幕")

    device.setBrightnessMode(1) // 自动亮度模式
    device.cancelKeepingAwake() // 取消设备常亮
    
    // Root
    // Power()

    // No Root
    press(Math.floor(device.width / 2), Math.floor(device.height * 0.973), 1000) // 小米的快捷手势：长按Home键锁屏
}


// ===================== 功能函数 =======================

function dateDigitToString(num){
    return num < 10 ? '0' + num : num
}

function getCurrentTime(){
    var currentDate = new Date()
    var hours = dateDigitToString(currentDate.getHours())
    var minute = dateDigitToString(currentDate.getMinutes())
    var second = dateDigitToString(currentDate.getSeconds())
    var formattedTimeString = hours + ':' + minute + ':' + second
    return formattedTimeString
}

function getCurrentDate(){
    var currentDate = new Date()
    var year = dateDigitToString(currentDate.getFullYear())
    var month = dateDigitToString(currentDate.getMonth() + 1)
    var date = dateDigitToString(currentDate.getDate())
    var week = currentDate.getDay()
    var formattedDateString = year + '-' + month + '-' + date + '-' + WEEK_DAY[week]
    return formattedDateString
}

// 通知过滤器
function filterNotification(bundleId, abstract, text) {
    
    var check = BUNDLE_ID_WHITE_LIST.some(function(item) {return bundleId == item})
    
    if (!NOTIFICATIONS_FILTER || check) {
        console.verbose(bundleId)
        console.verbose(abstract)
        console.verbose(text)
        console.verbose("---------------------------")
        return true
    }
    else {
        return false 
    }
}

// 保存本地数据
function setStorageData(name, key, value) {
    const storage = storages.create(name)  // 创建storage对象
    storage.put(key, value)
}

// 读取本地数据
function getStorageData(name, key) {
    const storage = storages.create(name)
    if (storage.contains(key)) {
        return storage.get(key, "")
    }
    // 默认返回undefined
}

// 删除本地数据
function delStorageData(name, key) {
    const storage = storages.create(name)
    if (storage.contains(key)) {
        storage.remove(key)
    }
}
```

## 使用方法
### AutoJs
AutoJs是安卓平台上的JavaScript自动化工具 https://github.com/hyb1996/Auto.js

官方文档：https://hyb1996.github.io/AutoJs-Docs/#/?id=%E7%BB%BC%E8%BF%B0

下载：[Auto.js 4.1.1a Alpha2-armeabi-v7a-release](https://www.lanzous.com/i56aexi "Auto.js 4.1.1a Alpha2-armeabi-v7a-release")

PC和手机连接到同一网络，使用 VSCode + Auto.js插件（在扩展中心搜索 "hyb1996"） 就能调试脚本并保存到手机上

### Tasker
<img width="270" height="585" src="https://github.com/georgehuan1994/DingDing-Automatic-Clock-in/blob/master/图片/截图_004.jpg"/>

1. 添加一个 "通知" 操作任务，通知标题修改为 "定时打卡"，通知文字随意，通知优先级设为 1

2. 添加两个配置文件，使用日期和时间作为条件，分别在上班前和下班后触发

或者[下载任务和配置文件](https://github.com/georgehuan1994/DingDing-Automatic-Clock-in/tree/master/Tasker配置 "下载配置")，导入到Tasker中使用

#### 导入方法
1. 长按 菜单栏-任务，导入"发送通知.tsk.xml"。（在任务编辑界面左下方有一个三角形的播放按钮，点击即可发送通知，方便调试。）

2. 长按 菜单栏-配置文件，导入"上班打卡.prf.xml" 和 "下班打卡.prf.xml" 

### 远程打卡
- 回复标题为 "打卡" 的邮件，即可触发打卡进程

- 回复标题为 "考勤结果" 的邮件，即可查询最新一次打卡结果

### 暂停/恢复定时打卡
- 回复标题为 "暂停" 的邮件，即可暂停定时打卡功能（仅暂停定时打卡，不影响远程打卡功能）

- 回复标题为 "恢复" 的邮件，即可恢复定时打卡功能

## 注意事项
- 首次启动AutoJs时，需要为其开启无障碍权限

- 运行脚本前，请在AutoJs菜单栏中（从屏幕左边划出），开启“通知读取权限”

- AutoJs、Tasker可息屏运行，需要在系统设置中开启通知亮屏

- 为保证AutoJs、Tasker进程不被系统清理，可调整它们的电池管理策略、加入管理应用的白名单，为其开启前台服务、添加应用锁...

- 虽然脚本可执行完整的打卡步骤，但推荐开启钉钉的极速打卡功能，在钉钉启动时即可完成打卡，应把后续的步骤视为极速打卡失败后的保险措施

## 更新日志
### 2021-02-07
优化：防止监听事件被耗时操作阻塞。

### 2021-01-15

针对钉钉6.0版本进行调整：

1. 取消了 从消息界面进入工作台 以及 从工作台进入考勤打卡界面 这两个过程

2. 启动并成功登录钉钉后，直接使用URL Scheme拉起考勤打卡界面

### 2021-01-08

修复：通知过滤器报错

### 2020-12-30

优化：现在可以通过邮件来 暂停/恢复 定时打卡功能，以应对停工停产，或其他需要暂时停止定时打卡的特殊情况

### 2020-12-04

优化：打卡过程在子线程中执行，钉钉返回打卡结果后，直接中断子线程，减少无效操作

### 2020-10-27

修复：当钉钉的通知文本为null时，indexOf()方法无法正常执行

### 2020-09-24

优化：若找不到考勤按钮，则使用URL Scheme直接拉起考勤打卡界面

```javascript
function attendKaoqin(){
    var a = app.intent({
        action: "VIEW",
        data: "dingtalk://dingtalkclient/page/link?url=https://attend.dingtalk.com/attend/index.html"
      });
      app.startActivity(a);
      sleep(5000)
}
```

#### 获取URL的方式如下：

1. 在PC端找到 “智能工作助理” 联系人

2. 发送消息 “打卡” ，点击 “立即打卡” 

3. 弹出一个二维码。此二维码就是拉起考勤打卡界面的 URL，用自带的相机或其他应用扫描，并在浏览器中打开，即可获得完整URL

4. 观察获取到的URL，找到 `CorpId=xxxxxxxxxxxxxxxxxxx` ，将CorpId的值填写到的脚本开头的CORP_ID这个常量中

5. 仅使用 `dingtalk://dingtalkclient/page/link?url=https://attend.dingtalk.com/attend/index.html`，也可以拉起旧版打卡界面，钉钉会自动获取企业的CorpId。如果加入了多个组织，且没有填写CorpId，则在拉起考勤界面时会弹出一个选择组织的对话框。

### 2020-09-11

1. 将上次考勤结果储存在本地

2. 将运行日志储存在本地 /sdcard/脚本/Archive/

3. 修复在下班极速打卡之后，重复打卡的问题

### 2020-09-04

将 "打卡" 与 "发送邮件" 分离成两个过程，打卡完成后，将钉钉返回的考勤结果作为邮件正文发送

### 2020-09-02

钉钉工作台界面改版（新增考勤打卡的快捷入口）。无法通过 "考勤打卡" 相关属性获取控件，改为使用 "去打卡" 文本获取按钮。若找不到 "去打卡" 按钮，则直接点击 "考勤打卡" 的屏幕坐标

## 声明

此仓库及脚本仅供学习交流，欢迎转载。旨在让人们关注996制度的存在和非法性，并尝试改变这种现象。

根据1994年第八届全国人大常委会通过和2018年第十三届全国人大常委会修正的《中华人民共和国劳动法》规定，劳动者**每日工作时间不超过8小时，平均每周工作时间不超过44小时，而996工作制每周至少要工作72个小时**，远超法律标准，因此996工作制度违反劳动法。

而钉钉却允许企业管理者违反法律，非法排班！  

<blockquote>

**第三十六条**　国家实行劳动者每日工作时间不超过八小时、平均每周工作时间不超过四十四小时的工时制度。

**第四十一条**　用人单位由于生产经营需要，经与工会和劳动者协商后可以延长工作时间，一般每日不得超过一小时；因特殊原因需要延长工作时间的，在保障劳动者身体健康的条件下延长工作时间每日不得超过三小时，但是每月不得超过三十六小时。

**第四十四条**　有下列情形之一的，用人单位应当按照下列标准支付高于劳动者正常工作时间工资的工资报酬：

（一）安排劳动者延长工作时间的，支付不低于工资的百分之一百五十的工资报酬；  
（二）休息日安排劳动者工作又不能安排补休的，支付不低于工资的百分之二百的工资报酬；  
（三）法定休假日安排劳动者工作的，支付不低于工资的百分之三百的工资报酬。  

**第九十条**　用人单位违反本法规定，延长劳动者工作时间的，由劳动行政部门给予警告，责令改正，并可以处以罚款。

**第九十一条**　用人单位有下列侵害劳动者合法权益情形之一的，由劳动行政部门责令支付劳动者的工资报酬、经济补偿，并可以责令支付赔偿金：

（二）拒不支付劳动者延长工作时间工资报酬的；

</blockquote>

---

    如果觉得还不错的话，就点击右上角, 给我个Star ⭐️ 鼓励一下我吧~
