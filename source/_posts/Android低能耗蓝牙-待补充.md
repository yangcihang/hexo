---
title: Android低能耗蓝牙(待补充)
date: 2017-10-21 23:17:27
tags: Android
categories: Android
---

BLE分为三部分Service、Characteristic、Descriptor，这三部分都由UUID作为唯一标示符。一个蓝牙4.0的终端可以包含多个Service，一个Service可以包含多个Characteristic，一个Characteristic包含一个Value和多个Descriptor，一个Descriptor包含一个Value。一般来说，Characteristic是手机与BLE终端交换数据的关键。另外，这些UUID都写在硬件里，我们通过BLE提供的API可以读取到。<!--more-->

## 大体流程

- 打开蓝牙
- 扫描设备
- 连接设备
- 写入数据，控制设备
- 读取设备返回的数据
- 断开设备

## 不一样的蓝牙搜索
在传统的蓝牙上，我们使用BluetoothAdapter.startDiscovery的方法来进行蓝牙扫描。**BluetoothAdapter.startDiscovery在大多数手机上是可以同时发现经典蓝牙和Ble的，但是startDiscovery的回调无法返回Ble的广播，所以无法通过广播识别设备，且startDiscovery扫描Ble的效率比StartLeScan低很多。**所以在实际应用中，还是StartDiscovery和StartLeScan分开扫，前者扫传统蓝牙，后者扫低功耗蓝牙。因此本篇文章的扫描用StartLeScan方法来进行扫描的。

## 蓝牙扫描

我们还得继续用到我们的老朋友BluetoothAdapter。首先打开蓝牙，这步操作和上篇文章一样，不必多说。主要是蓝牙扫描这一过程。首先，调用adapter的startLeScan(LeScanCallback callback)或者是startLeScan(final UUID[] serviceUuids, final LeScanCallback callback)方法，这两种方法的区别是前者没有指定UUID，后者是进行精准扫描，根据UUID来进行匹配。其中我们需要一个叫LeScanCallback的这一参数。这个回调接口需要我们自己去实现，主要的功能是实现扫描时的监听，类似传统方式用广播监听搜索时蓝牙设备的变化。对于这个callBack我们可以实现以下两个方法:

```
new ScanDeviceCallback() {
            @Override
            public void onScanFinish() {
             ToastUtil.showToast("没有找到设备");
            }

            @Override
            public void onLeScan(BluetoothDevice device, int rssi, byte[] scanRecord) {
                 ToastUtil.showToast("扫描到设备-------", device.getName());
            }
        });
```
顾名思义，一个是搜索结束，一个是搜索过程中扫描到设备时回调。我们可以自己定义一个扫描时间，用handler进行延迟处理，当扫描结束时，调用scanDeviceCallback.onScanFinish()和bluetoothAdapter.stopLeScan(scanDeviceCallback)方法来结束扫描。**这里还是要提醒一下，蓝牙设备的扫描是很耗费资源的，一定要手动停止。**

## 蓝牙连接和交互
- 首先，我们需要获取到BluetoothDevice，和传统蓝牙蓝牙获取到远程设备的方式一样，都是通过在监听回调时获取到BluetoothDevice，如果得到BluetoothDevice的地址，也可以通过adapter.getRemoteDevice(device.getAddress())这种方式来获取。
- 接下来是蓝牙连接的最重要的一步，连接蓝牙设备。我们需要创建一个BluetoothGatt对象，然后通过device的connectGatt(Context context, boolean autoConnect,BluetoothGattCallback callback）方法返回一个BluetoothGatt对象。第一个参数传入一个context，第二个参数是是否自动连接，一般传入false，第三个参数又是一个callback，这个callback监听的是连接时的状态。一般情况下重写这么几个函数:

```
private BluetoothGattCallback bluetoothGattCallback = new BluetoothGattCallback() {
        @Override
        public void onConnectionStateChange(BluetoothGatt gatt, int status, int newState) {
           ToastUtil.showToast("连接状态发生改变");
            if (status != BluetoothProfile.STATE_DISCONNECTED) {
                gatt.disconnect();
                gatt.close();
            } else if (newState == BluetoothProfile.STATE_CONNECTED) {
                gatt.discoverServices();
                ToastUtil.showToast("开始发现服务");
            } else {
                gatt.disconnect();
                gatt.close();
            }

        }

        @Override
        public void onServicesDiscovered(BluetoothGatt gatt, int status) {
            ToastUtil.showToast("服务已经发现);
        }


        @Override
        public void onCharacteristicWrite(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic, int status) {
        //在此写入数据
            if (status == BluetoothGatt.GATT_SUCCESS) {
                ToastUtil.showToast("写入数据");
            }
        }

        @Override
        public void onCharacteristicChanged(BluetoothGatt gatt, BluetoothGattCharacteristic characteristic) {
        //在此可以接受到数据
            ToastUtil.showToast( "接受数据");
        }
    };
```
- 当连接完成后，我们可以在这个callback里面监听到各种连接状况。我们可以看到其中有一个BluetoothGattCharacteristic参数。这个参数是干什么用呢？这个参数极其重要，Characteristic是手机与BLE终端交换数据的关键。我们可以在上述回调的这一步```onServicesDiscovered(BluetoothGatt gatt, int status){
 }```来实例化这个Characteristic,具体方式如下:
 
 ```
 @Override
        public void onServicesDiscovered(BluetoothGatt gatt, int status) {
            ToastUtil.showToast("服务已经发现");
            if (status == BluetoothGatt.GATT_SUCCESS) {
                bluetoothGattCharacteristic = gatt.getService(Constant.UUID_SERVICE).getCharacteristic(Constant.UUID_CHAR);
                bluetoothGatt.setCharacteristicNotification(bluetoothGattCharacteristic, true);
                ToastUtil.showToast("设置Notification成功");
            }
        }
 ```
 BluetoothGattCharacteristic实例是使用gatt.getService(Constant.UUID_SERVICE).getCharacteristic(Constant.UUID_CHAR);来获取当前的需要用到的Characteristic。注意，第一个方法getService(UUID uuid)返回的是一个service，service概况来说是一系列Characteristic的集合，我们通过设备指定的UUID才能获取到设备的service。然后再通过每一个Characteristic的UUID获取到这一特征属性的对象。举个例子：我们通过getService方法和UUID获取到了小米手环的service，然后又通过getCharacteristic方法和UUID获取到小米手环测量心率的一个特征。然后通过BluetoothGatt来设置Characteristic可通知，即我们可以接受到设备的写入操作。
- 数据的交互  当我们获取到Characteristic后，我们就可以和设备进行交互了。交互方式举例如下:

	```
	bluetoothGattCharacteristic.setValue(byte[]);
	bluetoothGatt.writeCharacteristic(bluetoothGattCharacteristic);
	```
通过Characteristic的setValue方法和gatt的writeCharacteristic方法写入，然后就会触发到BluetoothGattCallback中的onCharacteristicWrite回调。
读取数据也是从BluetoothGattCallback的onCharacteristicChanged方法来获取到设备通知的数据，也是通过Characteristic来获取到data的。
