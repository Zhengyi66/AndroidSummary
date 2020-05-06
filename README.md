# AndroidSummary
Android知识点总结

## 性能优化：
   1、过度绘制
   2、monkey使用
   3、BlockCanary使用
## 速度追踪
   
## 其他
   1、蓝牙通讯。
   https://blog.csdn.net/weixin_39079048/article/details/78923669
   https://blog.csdn.net/VNanyesheshou/article/details/51554852

NestedScrollingChildHelper



一、性能分析工具：
1、	Systrace
2、	Traceview方法调用跟踪
3、	GPU过度绘制
4、	GPU呈现模式分析
5、	Tracer for OpenGL ES Perspective
6、	Freqdump
7、	Oprofile

二、内存分析工具介绍
1、	Allocation Tracker
2、	heap update
3、	Memory Analyzer Tool(MAT)
4、	dumpsys meminfo
5、	procrank
6、	showmap
7、	DDMS Native Heap

三、常用功耗分析工具及方法介绍
1、top命令查看CPU占用率，分析与CPU相关的功耗问题
adb shell top -m 5
2、DUMPSYS 
adb shell  dumpsys alarm：列出当前有注册alarm的应用
adb shell  dumpsys power：列出Power Manager的参数，如wakelock时间等
adb shell  dumpsys batteryinfo：列出各功能使用power的状况。
3、cat内核节点查看系统状况
4、vmstat
使用vmstat命令可以得到关于CPU、进程、内存、内存分页、堵塞IO的信息

性能优化分类：
•	流畅 (卡顿)
•	稳定 (内存溢出、崩溃)
•	功耗	(耗电、流量、网络)
•	安装包 (包体积) 
