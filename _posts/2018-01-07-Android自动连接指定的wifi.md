---
layout:     post
title:      Android自动连接指定的wifi
subtitle:   不进行扫描操作，对指定的免密码WIFI进行连接
date:       2018-01-07
author:     Noah Zhang
header-img: img/post-bg-2018.jpg
catalog: true
tags:
- Android
- Kotlin
- 中文
---
# Android自动连接指定的wifi  
不进行扫描操作，对指定的免密码WIFI进行连接（之前没有连接过）,基于这个需求动手写了一个Demo，未连接成功时的状态，第一个参数输入SSID，第二个参数输入密码，密码可以根据实例情况输入，也可以不输入密码，因为有些Wifi免密码。  

Wifi连接管理类WifiConnector.kt，用Kotlin重写了：
```kotlin
package com.example.wificonnector

import android.net.wifi.WifiConfiguration
import android.net.wifi.WifiManager
import android.net.wifi.WifiConfiguration.KeyMgmt
import android.net.wifi.WifiConfiguration.AuthAlgorithm
import android.os.Handler
import android.os.Message
import android.text.TextUtils
import android.util.Log


class WifiConnector(internal var wifiManager: WifiManager) {
    internal var mHandler: Handler? = null

    /**
     * 向UI发送消息
     * @param info 消息
     */  
     fun sendMsg(info: String) {
        if (mHandler != null) {
            val msg = Message()
            msg.obj = info
            mHandler!!.sendMessage(msg)// 向Handler发送消息  
            
        } else {
            Log.e("wifi", info)
        }
    }

    //WIFICIPHER_WEP是WEP ，WIFICIPHER_WPA是WPA，WIFICIPHER_NOPASS没有密码
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
        for (existingConfig in existingConfigs) {
            if (existingConfig.SSID == "\"" + SSID + "\"") {
                return existingConfig
            }
        }
        return null
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
        // nopass
        if (Type == WifiCipherType.WIFICIPHER_NOPASS) {
            config.allowedKeyManagement.set(WifiConfiguration.KeyMgmt.NONE)
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
            config.allowedPairwiseCiphers
                    .set(WifiConfiguration.PairwiseCipher.TKIP)
            // 此处需要修改否则不能自动重联  
            // config.allowedProtocols.set(WifiConfiguration.Protocol.WPA)  
            config.allowedGroupCiphers.set(WifiConfiguration.GroupCipher.CCMP)
            config.allowedPairwiseCiphers
                    .set(WifiConfiguration.PairwiseCipher.CCMP)
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

    internal inner class ConnectRunnable(private val ssid: String, private val password: String, private val type: WifiCipherType) : Runnable {

        override fun run() {
            try {
                // 打开wifi  
                openWifi()
                sendMsg("opened")
                Thread.sleep(200)
                // 开启wifi功能需要一段时间(我在手机上测试一般需要1-3秒左右)，所以要等到wifi  
                // 状态变成WIFI_STATE_ENABLED的时候才能执行下面的语句  
                while (wifiManager.wifiState == WifiManager.WIFI_STATE_ENABLING) {
                    try {
                        // 为了避免程序一直while循环，让它睡个100毫秒检测……  
                        Thread.sleep(100)
                    } catch (ie: InterruptedException) {
                    }

                }

                val wifiConfig = createWifiInfo(ssid, password,
                        type)
                //
                if (wifiConfig == null) {
                    sendMsg("wifiConfig is null!")
                    return
                }

                val tempConfig = isExsits(ssid)

                if (tempConfig != null) {
                    wifiManager.removeNetwork(tempConfig.networkId)
                }

                val netID = wifiManager.addNetwork(wifiConfig)
                val enabled = wifiManager.enableNetwork(netID, true)
                sendMsg("enableNetwork status enable=" + enabled)
                val connected = wifiManager.reconnect()
                sendMsg("enableNetwork connected=" + connected)
                sendMsg("连接成功!")
            } catch (e: Exception) {
                // TODO: handle exception  
                sendMsg(e.message.toString())
                e.printStackTrace()
            }

        }
    }

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
}
```
#### 参考：<http://www.cnblogs.com/best/p/5634724.html>
