---
title: Android传统蓝牙
date: 2017-10-21 23:16:52
tags: Android
categories: Android
---

## 蓝牙建立连接的几个过程
- 打开蓝牙
- 查找附近已配对或可用设备
- 连接设备
- 设备间数据交换

## API及常用函数
- BluetoothAdapter:代表本地蓝牙适配器，是所有蓝牙的交互入口。使用这个可以发现其他蓝牙设备，查询配对设备和创建BluetoothServerSocket来为监听与其他设备通信<!--more-->
	- enable()直接开启蓝牙
	- getBondedDevices()返回一个set集合，类型为BluetoothDevice
	- getState()返回当前本地蓝牙适配器的状态
	- startDiscovery()开始搜索远程蓝牙
	- isDiscovering()返回是否在搜索的状态
	- isEnabled()返回蓝牙是否可用
	- listenUsingRfcommWithServiceRecord(String name,UUID uuid)返回一个BluetoothServerSocket,即创建一个BluetoothServerSocket来监听client的连接。
- BluetoothDevice 代表一个远程的蓝牙设备，可以使用这个来请求一个与远程设备的BluetoothSocket连接，也可以查看设备的名称地址之类的设备信息
	- createBond（）和远程蓝牙配对
	- getBondState()返回配对的状态
	- createRfcommSocketToServiceRecord（UUID uuid）创建一个BluetoothSocket连接
	- getName() getUuids() getType getAddress()字面意思
- BluetoothSocket 代表蓝牙的socket接口，它允许一个应用与其他蓝牙设备通过输入输出流来交换数据
	- connect() 和远程服务连接
	- getIputStream() 返回一个输入流
	- getOutputSteram() 返回一个输出流
	- getRemoteDevice() 返回远程连接的BluetoothDevice
- BluetoothServerSocket 代表一个服务器的socket，监听请求。连接两台设备时，必须有一台开启一个serverSocket。BluetoothServerSocket将返回一个已连接的BluetoothSocket，接受该连接。
	- accept()这是个阻塞的方法，直到有蓝牙和ServerSocket建立连接时，解除阻塞，返回一个BluetoothSocket。

## 蓝牙搜索、配对、广播接收
 1. 创建adapter  基本上都是使用
 ```bluetoothAdapter = BluetoothAdapter.getDefaultAdapter();```这样的方法来创建一个默认的适配器的。
 2. 打开蓝牙  打开蓝牙有三种方式：第一种是直接使用adapter.enable方法，这样是直接打开蓝牙的。第二种是用intent隐式打开
 
 ```
      Intent intent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);
      startActivityForResult(intent, 1);
 ```
第三种就是用户直接在手机打开。

 3. 扫描设备  使用adapter的startDiscovery()方法。这个方法是经典蓝牙的扫描方法。注意，这个方法很占用资源，扫描设备大概会持续10s，然后会自行关闭。但是若用户执行多次扫描时候，一定要记住使用cancelDiscovery方法将扫描关闭。
 4. 广播接收  在扫描设备时，系统会将扫描的状态广播出去,我们可以从广播中获取到搜索到的设备，即BluetoothDevice对象。所以我们需要写一个广播接收器来接收他们。基本操作如下：
 
 	```
 	  private BroadcastReceiver discoveryReceiver = new BroadcastReceiver() {

            @Override
            public void onReceive(Context context, Intent intent) {
                String action = intent.getAction();

                if (BluetoothAdapter.ACTION_DISCOVERY_STARTED.equals(action)) {
                    Log.i(TAG, "onReceive: 开始搜索");

                } else if (BluetoothDevice.ACTION_FOUND.equals(action)) {

                    BluetoothDevice bluetoothDevice = intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE);  //获取设备,发现远程蓝牙设备
                    String name = bluetoothDevice.getName();
                    String address = bluetoothDevice.getAddress();
                    Log.i(TAG, "onReceive: name=" + name + " address=" + address);
                    serversText.append(name + ":" + address + "\n");

                    discoveredDevices.add(bluetoothDevice);

                } else if (BluetoothAdapter.ACTION_DISCOVERY_FINISHED.equals(action)) {
                    Log.i(TAG, "onReceive:  搜索结束");

                    if (discoveredDevices.isEmpty()) {
                        Log.i(TAG, "onReceive:  未发现设备");
                    }
                }
            }
            if (BluetoothDevice.ACTION_BOND_STATE_CHANGED.equals(action)) {
                switch (device.getBondState()) {
                    case BluetoothDevice.BOND_BONDING:
                        ToastUtil.showToast("正在配对");
                        break;
                    case BluetoothDevice.BOND_BONDED:
                        ToastUtil.showToast("配对完成");
                        break;
                    case BluetoothDevice.BOND_NONE:
                        ToastUtil.showToast("取消配对");
                    default:
                        break;
                }
            }
    };
 	```
 代码中用了一个list来存放搜索到的设备信息，其中还有绑定的一些广播在内。然后再对广播进行注册：

 ```
 		 IntentFilter discoveryFilter = new IntentFilter();
       	 discoveryFilter.addAction(BluetoothAdapter.ACTION_DISCOVERY_STARTED);
       	 discoveryFilter.addAction(BluetoothAdapter.ACTION_DISCOVERY_FINISHED);
      	  discoveryFilter.addAction(BluetoothDevice.ACTION_FOUND);
        	registerReceiver(discoveryReceiver, discoveryFilter);
 ```

 5. 蓝牙绑定  一般情况下，如果两个设备需要通信首先得进行绑定，然后在广播接收器中监听其状态。不过在进行socket连接的时候，会自行访问用户是否进行设备绑定的。如果有必要的话，我们可以使用反射来进行连接，方法如下：
 
 ```
  Method createBondMethod = null;
            try {
                createBondMethod = BluetoothDevice.class.getMethod("createBond");
            } catch (NoSuchMethodException e) {
                e.printStackTrace();
            }
            ToastUtil.showToast("开始配对");
            try {
                returnValue = (Boolean) createBondMethod.invoke(btDev);
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (InvocationTargetException e) {
                e.printStackTrace();
            }
            if (returnValue) {
                ToastUtil.showToast("配对成功");

            }
        }
 ```
 解除绑定：
 
 ```
 if (device != null) {
            try {
                Method m = device.getClass().getMethod("removeBond", (Class[]) null);
                boolean returnValue = (boolean) m.invoke(device, (Object[]) null)；
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
 ```
 
 ## 设备间的通信
 下来就是进行socket通信了通信分为client模块和server模块和通信模块来说吧。

 ### server
 server模块主要功能就是实现一个设备的监听，当监听到有连接访问的操作时，返回一个BluetoothSocket，然后通过调用BluetoothSocket的输入输出流来进行两个设备间的通信。因此设备需要在一开始就开启一个线程去不断监听，在用Handler来将监听到的数据或是异常在主线程中展示给用户。server监听线程如下：
 
 ```
 /**
 * 服务器监听线程
 */
public class BluetoothServerConnThread extends Thread {

    private Handler serviceHandler;        //用于同Service通信的Handler
    private BluetoothAdapter adapter;
    private BluetoothSocket socket;        //用于通信的Socket
    private BluetoothServerSocket serverSocket; //serverSocket


    /**
     * 构造函数
     *
     * @param handler 在构造中传入需要处理数据的handler
     */
    public BluetoothServerConnThread(Handler handler, BluetoothAdapter bluetoothAdapter) {
        this.serviceHandler = handler;
        adapter = bluetoothAdapter;
    }

    @Override
    public void run() {

        try {
            if (serverSocket == null) {
                serverSocket = adapter.listenUsingRfcommWithServiceRecord("Server", Constant.PRIVATE_UUID);
            }
            //accept是一个阻塞的方法，返回一个socket连接
            socket = serverSocket.accept();

        } catch (Exception e) {
            //发送连接失败消息
            serviceHandler.obtainMessage(BluetoothTools.MESSAGE_CONNECT_ERROR).sendToTarget();
            e.printStackTrace();
            return;
        } finally {
            try {
                serverSocket.close();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

       /* if (socket.isConnected()) {
           ToastUtil.showToast("BluetoothServerConnThread.run 已建立与客户连接");
        }*/
        if (socket != null) {
            //发送连接成功消息，消息的obj字段为连接的socket
            Message msg = serviceHandler.obtainMessage();
            msg.what = BluetoothTools.MESSAGE_CONNECT_SUCCESS;
            msg.obj = socket;
            msg.sendToTarget();
        } else {
            //发送连接失败消息
            serviceHandler.obtainMessage(BluetoothTools.MESSAGE_CONNECT_ERROR).sendToTarget();
            return;
        }
    }
}
```
 
 其中在serverSocket获取时，第二个参数为 Constant.PRIVATE_UUID，这个参数就是通信时候必要的UUID。只有UUID相同时，才能进行设备的通信。下来是客户端模块的操作。
 ### client
 客户端首先得去开启、搜索蓝牙设备，这个过程在上面已经说过了。下来主要是client如何与server来进行通信。首先我们需要获取到server端的蓝牙设备信息，这个操作在搜索蓝牙这步就已经能够得到了。搜索到后，调用BluetoothDevice的createRfcommSocketToServiceRecord（UUID uuid）方法来返回一个BluetoothSocket。然后调用socket的connect（）方法来连接到远程的server。在此之前，一定要记得关闭蓝牙的搜索。具体代码如下：
 
 ```
 /**
 * 蓝牙客户端连接线程
 */
public class BluetoothClientConnThread extends Thread {

    private Handler serviceHandler;        //用于向客户端Service回传消息的handler
    private BluetoothDevice serverDevice;    //服务器设备
    private BluetoothSocket socket;        //通信Socket

    private BluetoothAdapter mBluetoothAdapter;

    /**
     * 构造函数
     *
     * @param handler
     * @param serverDevice
     */
    public BluetoothClientConnThread(Handler handler, BluetoothDevice serverDevice, BluetoothAdapter mBluetoothAdapter) {
        this.serviceHandler = handler;
        this.serverDevice = serverDevice;

        this.mBluetoothAdapter = mBluetoothAdapter;
    }

    @Override
    public void run() {

        if (mBluetoothAdapter.isDiscovering()) { //判断当前是否正在搜索
            mBluetoothAdapter.cancelDiscovery();
        }

        if (serverDevice == null) {
            ToastUtil.showToast("没有获取到远程设备");
            return;
        }

        try {
            if (socket == null) {
                socket = serverDevice.createRfcommSocketToServiceRecord(BluetoothTools.PRIVATE_UUID);
                mBluetoothAdapter.cancelDiscovery();
            }
            socket.connect();


        } catch (Exception ex) {
            try {
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
            //发送连接失败消息
            serviceHandler.obtainMessage(BluetoothTools.MESSAGE_CONNECT_ERROR).sendToTarget();
            return;
        }

        //发送连接成功消息，消息的obj参数为连接的socket
        Message msg = serviceHandler.obtainMessage();
        msg.what = BluetoothTools.MESSAGE_CONNECT_SUCCESS;
        msg.obj = socket;
        msg.sendToTarget();
    }
}
 ```
 这样我们就建立好一个server和client的socket连接了，接下来我们需要做的就是通信了：

 ### socket通信
 通信方面比较简单，server和client都持有一个socket对象，我们只需要通过其输入输出流通信就可以了。因此我们可以也可以通过一个线程来实现，在server和client的handler处理message时，当传入的message消息为连接成功，我们就可以开启这个线程。通信的线程代码如下：
 
 ```
 /**
 * 蓝牙通讯线程
 */
public class BluetoothCommunThread extends Thread {

    private Handler serviceHandler;        //与Service通信的Handler
    private BluetoothSocket socket;
    private ObjectInputStream inStream;        //对象输入流
    private ObjectOutputStream outStream;    //对象输出流
    public volatile boolean isRun = true;    //运行标志位

    /**
     * 构造函数
     *
     * @param handler 用于接收消息
     * @param socket
     */
    public BluetoothCommunThread(Handler handler, BluetoothSocket socket) {
        this.serviceHandler = handler;
        this.socket = socket;
        try {
            this.outStream = new ObjectOutputStream(socket.getOutputStream());
            this.inStream = new ObjectInputStream(new BufferedInputStream(socket.getInputStream()));
        } catch (Exception e) {
            try {
                if (socket != null) {
                    socket.close();
                } else {
                    ToastUtil.showToast("BluetoothCommunThread.BluetoothCommunThread ddd socket is null");
                }

            } catch (IOException e1) {
                e1.printStackTrace();
            }
            //发送连接失败消息
            serviceHandler.obtainMessage(BluetoothTools.MESSAGE_CONNECT_ERROR).sendToTarget();
            e.printStackTrace();
        }
    }

    /**
     * 写入一个可序列化的对象
     */
    public void writeObject(Object obj) {
        if (obj != null) {
            try {
                outStream.flush();
                outStream.writeObject(obj);
                outStream.flush();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

    }

    @Override
    public void run() {

        while (true) {

            if (!isRun || Thread.currentThread().isInterrupted()) {
                ToastUtil.showToast("BluetoothCommunThread.run 线程已中断");
                break;
            }


            try {
             ToastUtil.showToast("BluetoothCommunThread.run inStream=" + inStream);
               /* if (inStream == null) {
                    break;
                }*/

                Object obj = inStream.readObject();

                System.out.println("BluetoothCommunThread.run ddd");
                //发送成功读取到对象的消息，消息的obj参数为读取到的对象
                Message msg = serviceHandler.obtainMessage();
                msg.what = BluetoothTools.MESSAGE_READ_OBJECT;
                msg.obj = obj;
                msg.sendToTarget();


            } catch (Exception ex) {
                //发送连接失败消息
                serviceHandler.obtainMessage(BluetoothTools.MESSAGE_CONNECT_ERROR).sendToTarget();
                ex.printStackTrace();
                return;
            }
        }


        //关闭流
        if (inStream != null) {
            try {
                inStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if (outStream != null) {
            try {
                outStream.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        if (socket != null) {
            try {
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }


}
```
 
 在线程中我们使用了一个阻塞的方法不断的去获取输出的信息。如果当我们连个设备间要停止通信时，调用上面取消配对所述的方法即可。 这样我们就实现了传统蓝牙间的简单通信的流程