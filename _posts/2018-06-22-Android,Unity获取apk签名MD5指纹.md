---
layout: post
title:  "Android,Unity获取apk签名MD5指纹"
date:   2018-6-22 16:31:54 +0800
categories: Blog
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    [Android,Unity]
---

# Android,Unity获取apk签名MD5指纹

## Android签名问题

Android平台的游戏包大多数都遇到过被破解和二次打包的问题，android平台本身的开放性，对于这种破解和二次打包的问题并没有提过多少有效的防范措施。

大多数游戏加密和防破解功能只能由开发者自行解决，apk签名验证就是一个比较有效的方案。在客户端获取到apk的签名指纹，提交到服务器进行比对，如果不是自己的签名，可以对客户端做各种功能限制。

## 命令行获取签名指纹

使用keytool 命令获取apk文件（app.apk）的签名指纹，输出的信息总包括md5指纹和ras指纹。

```shell
keytool -list -printcert -jarfile app.apk
```

keytool工具,使用keytool工具获取签名文件(.keystone文件) 的签名指纹。

```shell
keytool -list -v -keystore key.keystone
```
 
## Android平台javad代码获取apk签名md5指纹

把获取签名指纹的代码放入到UnityActivity中。游戏启动后，GetSignMd5Str() 获得签名指纹之后，通过Unity的接口传入到Unity中进行验证。

但是Java层的代码很容易反编译，破解者容易修改Java代码中的签名获得函数部分的代码，或者修改签名指纹传入处的代码。

```java
/**
 * 获取签名的MD5 指纹
 * @return
 */
public String GetSignMd5Str() {
    try {
        PackageInfo packageInfo = mActivity.getPackageManager().getPackageInfo(mActivity.getPackageName(), PackageManager.GET_SIGNATURES);
        Signature[] signs = packageInfo.signatures;
        Signature sign = signs[0];
        String signStr = EncryptionMD5(sign.toByteArray());

        signStr=signStr.toUpperCase();
        StringBuilder sb = new StringBuilder();
        for(int i=0;i<signStr.length();++i)
        {
            if(i>0 && i %2 ==0)
            {
                sb.append(':');
            }
            sb.append(signStr.charAt(i));
        }

        return sb.toString();
    } catch (PackageManager.NameNotFoundException e) {
        e.printStackTrace();
    }
    return "";
}


/**
 * MD5 加密
 * @param byteStr
 * @return
 */
public static String EncryptionMD5(byte[] byteStr) {
    MessageDigest messageDigest = null;
    StringBuffer md5StrBuff = new StringBuffer();
    try {
        messageDigest = MessageDigest.getInstance("MD5");
        messageDigest.reset();
        messageDigest.update(byteStr);
        byte[] byteArray = messageDigest.digest();
        for (int i = 0; i < byteArray.length; i++) {
            if (Integer.toHexString(0xFF & byteArray[i]).length() == 1) {
                md5StrBuff.append("0").append(Integer.toHexString(0xFF & byteArray[i]));
            } else {
                md5StrBuff.append(Integer.toHexString(0xFF & byteArray[i]));
            }
        }
    } catch (NoSuchAlgorithmException e) {
        e.printStackTrace();
    }
    return md5StrBuff.toString();
}
    
```

## 在Unity C# 代码里面获取APK签名

相比在java代码里面获取签名指纹，在unity里面获取签名指纹安全度更高。在Unity中直接调用android 的api,对其android的签名信息，生成签名签名指纹。Unity项目尽量使用IL2CPP的方式编译，所有的C#代码都会编译成C++代码，最后生成.o文件在android端运行。很大程度上加大破解的难度。


```c#

/// <summary>
/// 获取Android APK 的MD5 签名指纹
/// </summary>
/// <returns>The signature M d5 hash.</returns>
private  string GetSignatureMD5Hash()
{
	//Debug.Log ("GetSignatureMD5Hash");
	var player = new AndroidJavaClass("com.unity3d.player.UnityPlayer");
	var activity = player.GetStatic<AndroidJavaObject>("currentActivity");
	var PackageManager = new AndroidJavaClass("android.content.pm.PackageManager");

	var packageName = activity.Call<string>("getPackageName");

	var GET_SIGNATURES = PackageManager.GetStatic<int>("GET_SIGNATURES");
	var packageManager = activity.Call<AndroidJavaObject>("getPackageManager");
	var packageInfo = packageManager.Call<AndroidJavaObject>("getPackageInfo", packageName, GET_SIGNATURES);
	var signatures = packageInfo.Get<AndroidJavaObject[]>("signatures");
	if(signatures != null && signatures.Length > 0)
	{
		byte[] bytes = signatures[0].Call<byte[]>("toByteArray");

		var md5String = GetMD5Hash(bytes);
		md5String = md5String.ToUpper ();

		StringBuilder sb = new StringBuilder();
		for (int i = 0; i < md5String.Length; ++i) {
			if(i>0 && i %2 ==0)
			{
				sb.Append(':');
			}
			sb.Append (md5String[i]);
		}

		return sb.ToString ();

	}

	return null;
}

private  string GetMD5Hash(byte[] bytedata)
{
	//Debug.Log ("GetMD5Hash");
	try
	{ 
		System.Security.Cryptography.MD5 md5 = new System.Security.Cryptography.MD5CryptoServiceProvider();
		byte[] retVal = md5.ComputeHash(bytedata);



		StringBuilder sb = new StringBuilder();
		for (int i = 0; i < retVal.Length; i++)
		{
			sb.Append(retVal[i].ToString("x2"));
		}
		return sb.ToString();
	}
	catch (Exception ex)
	{
		throw new Exception("GetMD5Hash() fail,error:" + ex.Message);
	}
}

```

## THE END

签名验证很大程度解决了二次打包破解的问题，但是还有很多破击方式没法处理，比如钩子hook破解，内存修改....