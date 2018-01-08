---
layout:     post
title:      Android 自动连接指定的 Wifi
subtitle:   不进行扫描操作，对指定的免密码 WIFI 进行连接
date:       2018-01-07
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Android
- Kotlin
- 中文
---
# Android 自动连接指定的 Wifi  
不进行扫描操作，对指定的免密码 WIFI 进行连接（之前没有连接过）,基于这个需求动手写了一个 Demo，未连接成功时的状态，第一个参数输入 SSID，第二个参数输入密码，密码可以根据实例情况输入，也可以不输入密码，因为有些 Wifi 免密码。

## 权限
```xml
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE"/>
<uses-permission android:name="android.permission.INTERNET" />
```

## WifiAutoConnectManager.kt
Wifi 连接管理类 WifiAutoConnectManager.kt，用 Kotlin 重写了：
```kotlin
package com.example.wifiAutoConnectManager

import android.content.Context
import android.net.wifi.WifiConfiguration
import android.net.wifi.WifiConfiguration.AuthAlgorithm
import android.net.wifi.WifiConfiguration.KeyMgmt
import android.net.wifi.WifiManager
import android.text.TextUtils
import android.util.Log

class WifiAutoConnectManager(internal var wifiManager: WifiManager) {

    // 定义几种加密方式，一种是WEP，一种是WPA，还有没有密码的情况  
    
    enum class WifiCipherType {
        WIFICIPHER_WEP, WIFICIPHER_WPA, WIFICIPHER_NOPASS, WIFICIPHER_INVALID
    }

    // 提供一个外部接口，传入要连接的无线网  
    
    fun connect(ssid: String, password: String, type: WifiCipherType) {
        val thread = Thread(ConnectRunnable(ssid, password, type))
        thread.start()
    }

    // 查看以前是否也配置过这个网络  
    
    private fun isExsits(SSID: String): WifiConfiguration? {
        val existingConfigs = wifiManager
                .configuredNetworks
        return existingConfigs.firstOrNull { it.SSID == "\"" + SSID + "\"" }
    }

    private fun createWifiInfo(SSID: String, Password: String,
                               Type: WifiCipherType): WifiConfiguration {
        val config = WifiConfiguration()
        config.allowedAuthAlgorithms.clear()
        config.allowedGroupCiphers.clear()
        config.allowedKeyManagement.clear()
        config.allowedPairwiseCiphers.clear()
        config.allowedProtocols.clear()
        config.SSID = "\"" + SSID + "\""
        // config.SSID = SSID  
        
        // nopass  
        
        if (Type == WifiCipherType.WIFICIPHER_NOPASS) {
            // config.wepKeys[0] = ""  
            
            config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.NONE)
            // config.wepTxKeyIndex = 0  
            
        }
        // wep  
        
        if (Type == WifiCipherType.WIFICIPHER_WEP) {
            if (!TextUtils.isEmpty(Password)) {
                if (isHexWepKey(Password)) {
                    config.wepKeys[0] = Password
                } else {
                    config.wepKeys[0] = "\"" + Password + "\""
                }
            }
            config.allowedAuthAlgorithms.set(AuthAlgorithm.OPEN)
            config.allowedAuthAlgorithms.set(AuthAlgorithm.SHARED)
            config.allowedKeyManagement.set(KeyMgmt.NONE)
            config.wepTxKeyIndex = 0
        }
        // wpa  
        
        if (Type == WifiCipherType.WIFICIPHER_WPA) {
            config.preSharedKey = "\"" + Password + "\""
            config.hiddenSSID = true
            config.allowedAuthAlgorithms
                    .set(WifiConfiguration.AuthAlgorithm.OPEN)
            config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.TKIP)
            config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.WPA_PSK)
            config.allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.TKIP)
            // 此处需要修改否则不能自动重联  
            
            // config.allowedProtocols.set(WifiConfiguration.Protocol.WPA)  
            
            config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.CCMP)
            config.allowedPairwiseCiphers.set(WifiConfiguration.PairwiseCipher.CCMP)
            config.status = WifiConfiguration.Status.ENABLED

        }
        return config
    }

    // 打开wifi功能  
    
    private fun openWifi(): Boolean {
        var bRet = true
        if (!wifiManager.isWifiEnabled) {
            bRet = wifiManager.setWifiEnabled(true)
        }
        return bRet
    }

    // 关闭WIFI  
    
    private fun closeWifi() {
        if (wifiManager.isWifiEnabled) {
            wifiManager.isWifiEnabled = false
        }
    }

    internal inner class ConnectRunnable(private val ssid: String, private val password: String, private val type: WifiCipherType) : Runnable {

        override fun run() {
            // 打开wifi  
            
            openWifi()
            // 开启wifi功能需要一段时间(我在手机上测试一般需要1-3秒左右)，所以要等到wifi  
            
            // 状态变成WIFI_STATE_ENABLED的时候才能执行下面的语句  
            
            while (wifiManager.wifiState == WifiManager.WIFI_STATE_ENABLING) {
                try {
                    // 为了避免程序一直while循环，让它睡个100毫秒检测……  
                    
                    Thread.sleep(100)
                } catch (ie: InterruptedException) {
                    Log.e(TAG, ie.toString())
                }

            }

            val tempConfig = isExsits(ssid)

            if (tempConfig != null) {
                // wifiManager.removeNetwork(tempConfig.networkId)  
                
                val b = wifiManager.enableNetwork(tempConfig.networkId, true)
            } else {
                val wifiConfig = createWifiInfo(ssid, password, type)
                if (wifiConfig == null) {
                    Log.d(TAG, "wifiConfig is null!")
                    return
                }

                val netID = wifiManager.addNetwork(wifiConfig)
                val enabled = wifiManager.enableNetwork(netID, true)
                Log.d(TAG, "enableNetwork status enable=" + enabled)
                val connected = wifiManager.reconnect()
                Log.d(TAG, "enableNetwork connected=" + connected)
            }

        }
    }

    companion object {

        private val TAG = WifiAutoConnectManager::class.java.simpleName

        private fun isHexWepKey(wepKey: String): Boolean {
            val len = wepKey.length

            // WEP-40, WEP-104, and some vendors using 256-bit WEP (WEP-232?)  
            
            return if (len != 10 && len != 26 && len != 58) {
                false
            } else isHex(wepKey)

        }

        private fun isHex(key: String): Boolean {
            for (i in key.length - 1 downTo 0) {
                val c = key[i]
                if (!(c >= '0' && c <= '9' || c >= 'A' && c <= 'F' || c >= 'a' && c <= 'f')) {
                    return false
                }
            }

            return true
        }

        // 获取ssid的加密方式  

        fun getCipherType(context: Context, ssid: String): WifiCipherType {
            val wifiManager = context.applicationContext.getSystemService(Context.WIFI_SERVICE) as WifiManager
            val list = wifiManager.scanResults

            for (scResult in list) {
                if (!TextUtils.isEmpty(scResult.SSID) && scResult.SSID == ssid) {
                    val capabilities = scResult.capabilities
                    // Log.i("hefeng","capabilities=" + capabilities)  

                    if (!TextUtils.isEmpty(capabilities)) {

                        if (capabilities.contains("WPA") || capabilities.contains("wpa")) {
                            Log.i("hefeng", "wpa")
                            return WifiCipherType.WIFICIPHER_WPA
                        } else if (capabilities.contains("WEP") || capabilities.contains("wep")) {
                            Log.i("hefeng", "wep")
                            return WifiCipherType.WIFICIPHER_WEP
                        } else {
                            Log.i("hefeng", "no")
                            return WifiCipherType.WIFICIPHER_NOPASS
                        }
                    }
                }
            }
            return WifiCipherType.WIFICIPHER_INVALID
        }
    }
}
```

## MainActivity.kt 片段
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
        val wifiManager: WifiManager
        val wifiAutoConnectorManager: WifiAutoConnectManager  
        ...  
        wifiManager = getSystemService(Context.WIFI_SERVICE) as WifiManager
        wifiAutoConnectorManager = WifiAutoConnectManager(wifiManager)
        wifiAutoConnectorManager.connect("ssid", "password",WifiAutoConnectManager.WifiCipherType.WIFICIPHER_WPA)
        ...
}
```

#### 参考：<http://www.cnblogs.com/best/p/5634724.html>
