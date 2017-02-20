---
layout: post
title:  Summary Learning notes of Android input
categories: 驱动杂记
date: 2017-02-17
---

***

## 简述

一般的，当获取到有输入事件时，比如按键按下弹起、鼠标移动、触摸屏滑动等等事件发生了，在驱动层简单的调用 `input_report_key()` 和 `input_sync()` 两个函数，即可将输入事件层层上报，直到 Android 的窗口终端进行“消费”。  

当然，Kernel 上报的输入事件是非常非常原始的，Android 窗口层要能“消费”这些事件，还需一个中间层的“翻译处理”，并且不同的事件，面向的“消费者”也不一样，还得需要中间层进行分发---这一切就类似邮局收发快递包裹一样。  

由于“翻译处理”、“分发”等工作量非常巨大，则中间层也将比 Kernel 层更复杂---其包括了输入管理者（InputManager）、原始事件集散点（EventHub）、原始事件读取加工者（InputReader）、输入事件分发者（InputDispatcher）、输入事件接收者（InputEventReceiver）。  

以下从 Android 到 Kernel 做一个简单的笔记。  


##  Android

先简述所言，先盗图一张：  

![](/assets/input/framework-input.png)  

其涉及到的文件及目录如下：  

  frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp  
  frameworks/base/core/jni/android_view_InputEventReceiver.cpp  

  frameworks/native/services/inputflinger/InputManager.cpp  
  frameworks/native/services/inputflinger/EventHub.cpp  
  frameworks/native/services/inputflinger/InputReader.cpp  
  frameworks/native/services/inputflinger/InputDispatcher.cpp    

#### InputManagerService

在这里，会向系统注册 Input 的 Native 函数，以及注册 java 层提供的策略回调函数，使得 c/c++ 和 java 能够进行双向的调用通信。  

	int register_android_server_InputManager(JNIEnv* env) {
	    int res = jniRegisterNativeMethods(env, "com/android/server/input/InputManagerService",
	            gInputManagerMethods, NELEM(gInputManagerMethods));
		//....
		// Callbacks
	    // TouchCalibration
	
	    FIND_CLASS(gTouchCalibrationClassInfo.clazz, "android/hardware/input/TouchCalibration");
	    gTouchCalibrationClassInfo.clazz = jclass(env->NewGlobalRef(gTouchCalibrationClassInfo.clazz));
	
	    GET_METHOD_ID(gTouchCalibrationClassInfo.getAffineTransform, gTouchCalibrationClassInfo.clazz,
	            "getAffineTransform", "()[F");
		//...	
	}  

#### InputManager

InputManager, 顾名思义就是管理者吧。他拥有两名员工，InputDispatcher 和 InputReader。他所做的管理工作就是，为他的这两名员工搭建工作环境，以及下达开工和停工的指令而已。  

	InputManager::InputManager(
	        const sp<EventHubInterface>& eventHub,
	        const sp<InputReaderPolicyInterface>& readerPolicy,
	        const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
	    mDispatcher = new InputDispatcher(dispatcherPolicy);
	    mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
	    initialize();
	}  

在上面的构造函数里创建 `InputDispatcher()` 和  `InputReader()` 的对象。

	void InputManager::initialize() {
	    mReaderThread = new InputReaderThread(mReader);
	    mDispatcherThread = new InputDispatcherThread(mDispatcher);
	}  

上面 `InputManager::initialize()` 为他的俩员工创造了两个工作线程。  

	status_t InputManager::start() {
	    status_t result = mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
	    if (result) {
	        ALOGE("Could not start InputDispatcher thread due to error %d.", result);
	        return result;
	    }
	
	    result = mReaderThread->run("InputReader", PRIORITY_URGENT_DISPLAY);
	    if (result) {
	        ALOGE("Could not start InputReader thread due to error %d.", result);
	
	        mDispatcherThread->requestExit();
	        return result;
	    }
	
	    return OK;
	}  

`InputManager::start()`下令干活；  

	status_t InputManager::stop() {
	    status_t result = mReaderThread->requestExitAndWait();
	    if (result) {
	        ALOGW("Could not stop InputReader thread due to error %d.", result);
	    }
	
	    result = mDispatcherThread->requestExitAndWait();
	    if (result) {
	        ALOGW("Could not stop InputDispatcher thread due to error %d.", result);
	    }
	
	    return OK;
	}  

下令停止工作。  

InputManager 的工作就如上所述了。  


#### InputReader

`InputReader` 做的工作比较的多，但最主要的是以下两件事情：分析处理原始事件、管理输入设备。  

`InputReader` 的工作场所（工作线程）就是 `InputManager` 为其创建的 `InputReaderThread` 线程，他将一直在该线程中工作和休息着（睡眠、唤醒、等待等待），其入口为：  

	bool InputReaderThread::threadLoop() {
	    mReader->loopOnce();                                                                                                                                     
	    return true;
	}  

	void InputReader::loopOnce() {
		size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);
		processEventsLocked(mEventBuffer, count);
		mQueuedListener->flush();
	}  

`getEvents()` 是 `EventHub()` 为其私人定制的接口函数。而 `processEventsLocked()` 则是处理各种输入事件的，这其中包括输入设备的增加和删除，以及设备本身上报的事件。最后的 `flush()` 则是刷入到队列缓冲区，将所有的事件都交付给 `InputDispatcher` 进行分发。    

###### Events Processed  

	void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
		for (const RawEvent* rawEvent = rawEvents; count;) {
			if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
				processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
			} else {
	            switch (rawEvent->type) {                                                                                                                        
	            case EventHubInterface::DEVICE_ADDED:
	                addDeviceLocked(rawEvent->when, rawEvent->deviceId);
	                break;
	            case EventHubInterface::DEVICE_REMOVED:
	                removeDeviceLocked(rawEvent->when, rawEvent->deviceId);
	                break;
	            case EventHubInterface::FINISHED_DEVICE_SCAN:
	                handleConfigurationChangedLocked(rawEvent->when);
	                break;
	            default:
	                ALOG_ASSERT(false); // can't happen
	                break;
	            }
			}
		}
	}  

上面的 `for` 循环中就两个分支，一个是处理输入事件的，一个是管理输入设备的。  

####### Input Device Manage  

设备管理有以下功能：  

	DEVICE_ADDED			// 设备添加
	DEVICE_REMOVED  		// 设备移除
	FINISHED_DEVICE_SCAN	//设备查询/浏览

提供的函数为：  

	void InputReader::addDeviceLocked(nsecs_t when, int32_t deviceId) {}  
	void InputReader::removeDeviceLocked(nsecs_t when, int32_t deviceId){}  
	void InputReader::handleConfigurationChangedLocked(nsecs_t when){}  

####### Raw Events proccessed 

	void InputReader::processEventsForDeviceLocked(int32_t deviceId,                                                                                             
	        const RawEvent* rawEvents, size_t count) {
			device->process(rawEvents, count);
		}

	void InputDevice::process(const RawEvent* rawEvents, size_t count) {
	                InputMapper* mapper = mMappers[i];
	                mapper->process(rawEvent);	
	}  

`InputMapper`，顾名思义，输入事件映射，也即是在这里，事件开始真正的分门别类了。但并不是在 `InputMapper` 这个里面进行的，而是在 `InputMapper` 里面提供了一个纯虚函数 `process()`， 通过 c++ 的多态，在 `InputMapper` 的子类实现了，然后通过基类的指针，指向了子类的 `process()`。  

	class InputMapper {
		virtual void process(const InputEvent& event) = 0;
	}  

子类的实现如下：  

	void SwitchInputMapper::process(const RawEvent* rawEvent){};
	void SwitchInputMapper::processSwitch(int32_t switchCode, int32_t switchValue){}
	void VibratorInputMapper::process(const RawEvent* rawEvent){}
	void KeyboardInputMapper::process(const RawEvent* rawEvent){}
	void CursorInputMapper::process(const RawEvent* rawEvent){}
	void RotaryEncoderInputMapper::process(const RawEvent* rawEvent){}
	void TouchInputMapper::process(const RawEvent* rawEvent){}

经过以上流程后，`InputReader` 再使用 `QueuedInputListener` 给定的接口 `void QueuedInputListener::flush(){}` flush 一把，就把上面 Map 后的事件交给 `InputDispatcher` 去处理了。  

#### InputDispatcher

`InputDispatcher` 的工作线程为 `InputDispatcherThread`，将开始分发事件。其将根据事件的类型，进行不同方向的分发。  

	bool InputDispatcherThread::threadLoop() {
	    mDispatcher->dispatchOnce();
	    return true;
	}  

	void InputDispatcher::dispatchOnce() {
		 dispatchOnceInnerLocked(&nextWakeupTime);
	}  
	
	void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
		switch (mPendingEvent->type) {
			case EventEntry::TYPE_CONFIGURATION_CHANGED:
			break;
			case EventEntry::TYPE_DEVICE_RESET:
			break;
			case EventEntry::TYPE_KEY:
			 done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);
			break;
			case EventEntry::TYPE_MOTION:
		        done = dispatchMotionLocked(currentTime, typedEntry,
	            &dropReason, nextWakeupTime);
			break;
		}
	}  

以上函数，可重点查看 `void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime)` 的 `switch` 实现。


#### EventHub

EventHub 算是一个对 Android Input framework 的一个支撑，一个基石。但它是专为 `InputReader` 服务的，它为 `InputReader` 提供了一个最重要的工具，那就是 `EventHub::getEvents()` 函数接口。通过该接口，底层上报的输入事件才会源源不断的提供给 `InputReader` 进行处理。  

EventHub 通过使用 inotify 、epoll、 piple 机制，对输入设备的变化以及输入事件进行监控和处理。  

	EventHub::EventHub(void) {
		mEpollFd = epoll_create(EPOLL_SIZE_HINT);
		mINotifyFd = inotify_init();
		result = pipe(wakeFds);
		//...
	}  

读取事件的接口为 `EventHub::getEvents()` ：  

	size_t EventHub::getEvents(int timeoutMillis, RawEvent* buffer, size_t bufferSize) {
		readNotifyLocked();
		int pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, timeoutMillis);
	}  

当然，getEvents 实现较为复杂，其中包括了对设备的打开、读取、关闭等等操作。  

## Kernel

依然先盗图一张：  

![](/assets/input/kernel-input.png)  

在编写输入事件驱动时，一般的调用这两个函数即可：  

	input_report_*();  
	input_sync();

其中的 input_report_*() 是一组函数，包括以下几个：   

	input_report_key();    
	input_report_rel();    
	input_report_abs();  
	input_report_ff_status();  
	input_report_switch();  

其实以上几个函数，都是调用了 `input_event()` 函数：  
	
	void input_event(struct input_dev *dev, unsigned int type, unsigned int code, int value);  

只是每个函数调用的时候，传入的 event-type 是不一样的。 event-type 包括以下几种：  

	#define EV_SYN          0x00                                                                                                  
	#define EV_KEY          0x01
	#define EV_REL          0x02
	#define EV_ABS          0x03
	#define EV_MSC          0x04
	#define EV_SW           0x05                                                                                                                                 
	#define EV_LED          0x11
	#define EV_SND          0x12                                                                                                  
	#define EV_REP          0x14 
	#define EV_FF           0x15                                                                                                  
	#define EV_PWR          0x16                                                                                                  
	#define EV_FF_STATUS        0x17  

它们的含义，在 kernel/Documentation 目录下的 event-codes.txt 文件有解释：  

* EV_SYN  用于事件间的分割标志。事件可能按时间或空间进行分割，就像在多点触摸协议中的例子  
* EV_KEY  用来描述键盘，按键或者类似键盘设备的状态变化  
* EV_REL  用来描述相对坐标轴上数值的变化，例如：鼠标向左方移动了5个单位  
* EV_ABS  用来描述相对坐标轴上数值的变化，例如：描述触摸屏上坐标的值  
* EV_MSC  当不能匹配现有的类型时，使用该类型进行描述  
* EV_SW   用来描述具备两种状态的输入开关  
* EV_LED  用于控制设备上的LED灯的开和关  
* EV_SND  用来给设备输出提示声音  
* EV_REP  用于可以自动重复的设备  
* EV_FF   用来给输入设备发送强制回馈命令  
* EV_PWR  特别用于电源开关的输入  
* EV_FF_STATUS  用于接收设备的强制反馈状态

Input 子系统在 kernel 也是分成了多层，分别包含了input核心层（input-core）、input设备（input-device）和input事件处理层（input-handler），详见 CSDN 另外一份资料：  

[ Linux input子系统分析之二：深入剖析input_handler、input_core、input_device](http://blog.csdn.net/yueqian_scut/article/details/48792939)

---
over