f9omstw benchmark (台灣期交所)
==============================

## 基本說明
* sourc code
  * [libfon9](https://github.com/fonwin/libfon9)
  * [f9omstw](https://github.com/fonwin/f9omstw)

* 測量的時間點:
  * [T0 Client before send](https://github.com/fonwin/f9omstw/blob/9f3ed8a/f9omsrc/OmsRcClient_UT.c#L331)
  * [T1 Server before parse(封包收到時間)](https://github.com/fonwin/f9omstw/blob/9f3ed8a/f9omsrc/OmsRcServerFunc.cpp#L194)
  * [T2 Server after first updated(Sending, Queuing, Reject...)](https://github.com/fonwin/f9omstw/blob/9f3ed8a/f9omstw/OmsBackend.cpp#L202)
  * `T0-` = T0(n) - T0(n-1) 表示 Client 連續送單之間的延遲.
  * `T1-` = T1(n) - T1(n-1) 表示 RcServer 處理連續下單之間的延遲.
  * `T2-` = T2(n) - T2(n-1) 表示 OmsCore 處理連續下單要求之間的延遲 (包含送出給期交所).
  * 有測試交易所成功回覆, 但沒有計算交易所回覆的時間

---------------------------------------
## [初次啟動](Startup.md)

## 測試指令
* Client 下單程式 & 下單指令:
```
~/devel/output/f9omstw/release/f9omsrc/OmsRcClient_UT -a "dn=localhost:6601|ClosedReopen=5" -u fonwin

關閉 Client 端 log
> lf 0

新單測試
> set  TwfNew BrkId=8610|Symbol=XJFL9|PosEff=O|Pri=8000|Qty=33|Side=B|IvacNo=10|TimeInForce=R
> send TwfNew 1 g1
> send TwfNew 10 g2
> send TwfNew 100 g3
> send TwfNew 1000 g4
> send TwfNew 10000 g5
> send TwfNew 100000 g6

刪單測試
> set  TwfChg Qty=0|BrkId=8610|Market=f|SessionId=N|OrdNo=A0000
> send TwfChg 1 d1
> send TwfChg 10 d2 +
> send TwfChg 100 d3 +
> send TwfChg 1000 d4 +
> send TwfChg 10000 d5 +
> send TwfChg 100000 d6 +

慢速測試: 60 筆, 每筆間隔 1000 ms
> send TwfNew 60/1000 s1
```

* 分析程式
`~/devel/output/f9omstw/release/f9omstw/OmsLogAnalyser ~/f9utwf/logs/yyyymmdd/omstw.log `

---------------------------------------
## 測試結果
### 設備及工具
* 硬體: HP ProLiant DL380p Gen8 / E5-2680 v2 @ 2.80GHz
* OS: Ubuntu 16.04.2 LTS
* gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.9)
### 執行環境
* OmsCoreByThread: [`Cpu=19`](./fon9cfg/MaPlugins.f9gv#L7)
* [`chrt -f 90 taskset -c 10,11,12,13,14,15,16,17,18 ./run.sh`](./srun.sh#L7)
### 20191118
* sourc code
  * [libfon9](https://github.com/fonwin/libfon9) (版本: 9a7ca3b)
  * [f9omstw](https://github.com/fonwin/f9omstw) (版本: 9f3ed8a)
* 時間單位: us(microsecond)
* 連線設定: ApCode=4|IsUseSymNum=N
* 統計結果 [Wait=Block](./logs.block/20191118/omstw.log.Summary.txt)
* 統計結果 [Wait=Busy](./logs.busy/20191118/omstw.log.Summary.txt)
* 記憶體用量(沒有成交, 程式 f9utw 沒結束):
  * cat /proc/pid/status
  * TwfNew  111111筆: VmPeak: 1598504 kB
  * +TwfChg 111111筆: VmPeak: 1666820 kB
  * +TwfNew 100萬 筆: VmPeak: 3091772 kB (共 TwfNew 1,111,291筆, TwfChg 111,111筆)
  * +TwfNew 100萬 筆: VmPeak: 3791588 kB (共 TwfNew 2,111,291筆, TwfChg 111,111筆)
  * +TwfNew 100萬 筆: VmPeak: 4647052 kB (共 TwfNew 3,111,291筆, TwfChg 111,111筆)
  * +TwfNew 100萬 筆: VmPeak: 5435696 kB (共 TwfNew 4,111,291筆, TwfChg 111,111筆)
  * +TwfNew 100萬 筆: VmPeak: 6225624 kB (共 TwfNew 5,111,291筆, TwfChg 111,111筆)
  * 由於有用到 [ObjSupplier機制](https://github.com/fonwin/f9omstw/blob/9f3ed8a/f9omstw/OmsRequestFactory.hpp#L87)
    所以記憶體的使用量不是成線性增長.
* 10萬筆, 大約 1 秒完成: 收單解析、下單打包、部分 send() 成功
  * 但因網路頻寬、SNDBUF 配置... 等因素, 下單打包的資料, 可能仍保留在程式的緩衝區裡面.
  * T2 = 最後一筆 OMS 打包完畢的時間.
  * 以 g6:100000(Wait=Block) 最後這筆來看
    * T0 = 20191118131233.361450
    * T1 = 20191118131233.361466 (T1 - T0 = 16 us)
    * T2 = 20191118131233.361480 (T2 - T1 = 14 us)
      * T2(g6:100000) - T0(g6:1)(Client打包第1筆的時間) 20191118131232.516714 = 0.844766 秒
    * OMS 收到回報的時間 = 20191118131237.580898 ( - T2 = 花費 4.219418 秒)
  * 以 g6:100000(Wait=Busy) 最後這筆來看
    * T0 = 20191118094036.909721
    * T1 = 20191118094036.909731 (T1 - T0 = 10 us)
    * T2 = 20191118094036.909732 (T2 - T1 =  1 us)
      * T2(g6:100000) - T0(g6:1)(Client打包第1筆的時間) 20191118094035.886503 = 1.023229 秒
    * OMS 收到回報的時間 = 20191118094041.076135 ( - T2 = 花費 4.166403 秒)
  * 測試環境 OMS 到「模擬交易所」之間的網路頻寬 = 100M (`ethtool xxx`)
* 由底下結果看來, 使用 `Wait=Busy` 與 `Wait=Block` 比較, 在大部分情況下, 可以明顯降低延遲.
```
* 10 筆:
  * Block: T2-T1|50%= 88|75%= 89|90%=107|99%= 107|99.9%= 107|99.99%= 107|Worst= 89| 89|107|
  * Busy:  T2-T1|50%=  9|75%= 14|90%= 15|99%=  15|99.9%=  15|99.99%=  15|Worst= 14| 15| 15|
* 100 筆:
  * Block: T2-T1|50%=  7|75%= 83|90%=107|99%= 127|99.9%= 127|99.99%= 127|Worst=126|127|127|
  * Busy:  T2-T1|50%=  3|75%=  3|90%=  6|99%=  15|99.9%=  15|99.99%=  15|Worst= 12| 12| 15|
* 1000 筆:
  * Block: T2-T1|50%=  6|75%=  6|90%=  9|99%=  91|99.9%= 96|99.99%=   96|Worst= 95| 95| 96|
  * Busy:  T2-T1|50%=  3|75%=  3|90%=  4|99%=  14|99.9%= 18|99.99%=   18|Worst= 16| 17| 18|
* 1萬筆:
  * Block: T2-T1|50%=  6|75%=  7|90%=  8|99%=  19|99.9%= 509|99.99%= 589|Worst=573|573|589|
  * Busy:  T2-T1|50%=  3|75%=  3|90%=  5|99%=  11|99.9%=  17|99.99%=  33|Worst= 20| 21| 33|
* 10萬筆:
  * Block: T2-T1|50%=  5|75%=  6|90%=  7|99%=1841|99.9%=4368|99.99%=4751|Worst=4766|4767|4767|
  * Busy:  T2-T1|50%=  2|75%=  3|90%=  4|99%=1564|99.9%=4079|99.99%=5026|Worst=5089|5098|5102|
* 1筆/每秒, 共60次
  * Block: T2-T1|50%=116|75%=120|90%=121|99%= 298|99.9%= 298|99.99%= 298|Worst=139|177|298|
  * Busy:  T2-T1|50%= 10|75%= 11|90%= 11|99%= 204|99.9%= 204|99.99%= 204|Worst= 11| 31|204|
* 1筆/0.5秒, 共60次
  * Block: T2-T1|50%=114|75%=118|90%=121|99%= 121|99.9%= 121|99.99%= 121|Worst=121|121|121|
  * Busy:  T2-T1|50%= 10|75%= 11|90%= 11|99%=  12|99.9%=  12|99.99%=  12|Worst= 11| 12| 12|
* 1筆/0.1秒, 共60次
  * Block: T2-T1|50%=114|75%=119|90%=120|99%= 134|99.9%= 134|99.99%= 134|Worst=121|133|134|
  * Busy:  T2-T1|50%= 10|75%= 10|90%= 10|99%=  23|99.9%=  23|99.99%=  23|Worst= 11| 13| 23|
```
---------------------------------------
## 測試結果探討
* 詳細的探討, 等有空時再說吧!
* OmsCore 使用 OmsCoreByThread
  * 收單 thread 與 OmsCore 在不同 thread.
  * 可設定 WaitPolicy: `Wait=Busy` 或 `Wait=Block`(使用 condition variable).
  * 可綁定 CPU: `Cpu=19`
  * 如果不使用 OmsCoreByThread, 改用 OmsCoreByMutex(尚未實現) 延遲會更低嗎?
* OmsCore 的效率: `T2-`
  * 包含 send() 呼叫.
* Rc協定收單 的效率: `T1-`
  * 包含 recv() 呼叫.
* OmsCoreByThread(Context switch): `T2-T1`
  * 如果使用網路加速 library(例如: OpenOnload), 會有什麼變化呢?
* CPU cache 的影響?

### Linux 的 低延遲
* 這裡只是基礎, 關於核心的調教, 還要更多的研究.
* Linux 啟動時使用 isolate 參數, 將 cpu core 保留給 OMS 使用.
  * sudo vi /etc/default/grub
    * GRUB_CMDLINE_LINUX_DEFAULT="isolcpus=10,11,12,13,14,15,16,17,18,19,30,31,32,33,34,35,36,37,38,39"
  * sudo update-grub
* Linux 設定 irqbalance
* 將 CPU 設定為高效能(關閉省電模式), 但若 IO 設定有使用 `Wait=Busy` 參數, 也可以考慮不調整 CPU 時脈.
* 請參考 [rt-cfg.sh](./rt-cfg.sh)
* 啟動時的優先權及綁定 cpu, 請參考 [srun.sh](./srun.sh)
  * 使用 chrt 設定優先權
  * 使用 taskset 設定使用的 cpu cores
    * `chrt -f 90 taskset -c 10,11,12,13,14,15,16,17,18 ./run.sh`
    * 若有錯誤訊息: `chrt: failed to set pid 0's policy: Operation not permitted`   
      則要先執行:   `sudo setcap  cap_sys_nice=eip  /usr/bin/chrt`
