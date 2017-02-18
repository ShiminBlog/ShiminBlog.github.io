---
layout: post
title:  Learning Summary notes of Android input
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

`InputDispatcher` 的工作线程为 `InputDispatcherThread`，将开始分发事件。  

(//TODO...)
---
over