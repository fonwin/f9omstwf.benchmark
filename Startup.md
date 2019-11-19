﻿f9omstwf.benchmark / Startup
============================

## 初次啟動
### 啟動前準備事項
* 建立 fon9cfg/fon9common.cfg
```
$LogFileFmt=./logs/{0:f+'L'}/fon9sys-{1:04}.log
$MemLock = Y
```

* 建立 fon9cfg/fon9local.cfg
```
$HostId = 221
```

* 建立 API 表格, 從 `~/devel/f9omstw/f9omsrc/forms` 複製, 或建立連結.
  * `ln -s ~/devel/f9omstw/f9omsrc/forms forms`

* 取得 tmpsave/P06 、 tmpsave/P08.10 、 tmpsave/P08.20 、 tmpsave/P07.10.F906 ...

### 使用「管理模式」啟動程式
* `~/devel/output/f9omstw/release/f9utw/f9utw --admin`
初次啟動執行底下指令, 載入必要的模組, 底下的設定都是立即生效:
```
# 建立必要的 Device(TcpServer,TcpClient,Dgram,FileIO) factory, 放在 DeviceFactoryPark=FpDevice
ss,Enabled=Y,EntryName=TcpServer,Args='Name=TcpServer|AddTo=FpDevice' /MaPlugins/DevTcps
ss,Enabled=Y,EntryName=TcpClient,Args='Name=TcpClient|AddTo=FpDevice' /MaPlugins/DevTcpc
ss,Enabled=Y,EntryName=Dgram,Args='Name=Dgram|AddTo=FpDevice'         /MaPlugins/DevDgram
ss,Enabled=Y,EntryName=FileIO,Args='Name=FileIO|AddTo=FpDevice'       /MaPlugins/DevFileIO

# 建立 io 管理員, DeviceFactoryPark=FpDevice; SessionFactoryPark=FpSession; 設定儲存在 fon9cfg/MaIo.f9gv
ss,Enabled=Y,EntryName=NamedIoManager,Args='Name=MaIo|DevFp=FpDevice|SesFp=FpSession|Cfg=MaIo.f9gv|SvcCfg="ThreadCount=2|Capacity=100"' /MaPlugins/MaIo

# 啟動 OMS, 支援的券商: 8610,8611,8612
ss,Enabled=Y,EntryName=UtwOmsCore,Args='BrkId=8610|BrkCount=3'  /MaPlugins/iOmsCore

# 啟動 Rc下單協定
ss,Enabled=Y,EntryName=RcSessionServer,Args='Name=OmsRcSvr|Desp=f9OmsRc Server|AuthMgr=MaAuth|AddTo=FpSession'                     /MaPlugins/iOmsRcSv
ss,Enabled=Y,EntryName=OmsRcServerAgent,Args='OmsCore=omstw|Cfg=$TxLang={zh} $include:forms/ApiAll.cfg|AddTo=FpSession/OmsRcSvr'   /MaPlugins/iOmsRcSvAgent
```

### 設定測試及管理員帳號(fonwin)、密碼、權限
```
ss,RoleId=admin,Flags=0 /MaAuth/UserSha256/fonwin
/MaAuth/UserSha256/fonwin repw Password(You can use a sentence.)

# 設定管理權限.
ss,HomePath=/ /MaAuth/PoAcl/admin
ss,Rights=xff /MaAuth/PoAcl/admin/'/'
ss,Rights=xff /MaAuth/PoAcl/admin/'/..'

# 設定交易權限: 可用櫃號=A, 不限流量.
ss,OrdTeams=A /MaAuth/OmsPoUserRights/admin

# 設定交易帳號: 管理員, 任意帳號, AllowAddReport(x10)
ss            /MaAuth/OmsPoIvList/admin
ss,Rights=x10 /MaAuth/OmsPoIvList/admin/*
```

### 建立 Rc下單 Server
```
# ----- 設定 -----
ss,Enabled=Y,Session=OmsRcSvr,Device=TcpServer,DeviceArgs=6601 /MaIo/^edit:Config/iOmsRcSv

# 查看設定及套用時的 SubmitId
gv /MaIo/^apply:Config

# 套用設定, 假設上一行指令的結果提示 SubmitId=1
/MaIo/^apply:Config submit 1
# ----- MaIo 套用後生效 -----
# ---------------------------
```

### 建立 OMS 的台灣期交所 TMP 連線
```
# 設定台灣證交所 FIX 連線, ApCode=4(期貨商委託/成交), 6(期貨商短格式委託/成交)
# DeviceArgs 可設定期交所模擬主機ip:port, 系統也會根據 P06,P07 自動加上 dn=dns:port;
ss,Enabled=Y,Session=TWF,SessionArgs='FcmId=279|SessionId=50|Pass=9999|ApCode=4|IsUseSymNum=N',Device=TcpClient,DeviceArgs='192.168.1.33:9000|Bind=49003|ReuseAddr=Y|ClosedReopen=5' /TaiFex_TEST/LineMgrG1/UtwFutNormal_io/^edit:Config/L50

# 查看設定及套用時的 SubmitId
gv /TaiFex_TEST/LineMgrG1/UtwFutNormal_io/^apply:Config

# 套用設定, 假設上一行指令的結果提示 SubmitId=1
/TaiFex_TEST/LineMgrG1/UtwFutNormal_io/^apply:Config submit 1
```
