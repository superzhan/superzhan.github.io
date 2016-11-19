---
layout: post
title:  "Unity使用WWW类调用Restful接口"
date:   2016-11-15 20:15:54 +0800
categories: Blog
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    - Unity
---

# 1 BEGIN

维基百科，自由的百科全书
>具象状态传输（英文：Representational State Transfer，简称REST）是Roy Thomas Fielding博士于2000年在他的博士论文 "Architectural Styles and the Design of Network-based Software Architectures" 中提出来的一种万维网软件架构风格。
   目前在三种主流的Web服务实现方案中，因为REST模式与复杂的SOAP和XML-RPC相比更加简洁，越来越多的web服务开始采用REST风格设计和实现。例如，Amazon.com提供接近REST风格的Web服务执行图书查询；雅虎提供的Web服务也是REST风格的。

 RESTful API是目前比较成熟的一套互联网应用程序的API设计理论。

# 2 简单的Get请求

基本http的GET请求需要在一个协程中new一个WWW类并传入准确的API地址，执行`		yield return www ` 就会等待服务器的返回结果，之后才开始执行以后的代码。 Get请求的API参数需要加入到API的地址中去，格式 `URL+?key1=value1&key2=value2`。

```
public class NewBehaviourScript : MonoBehaviour {

	// Use this for initialization
	void Start () {
		StartCoroutine (IEHttpGet());
	
	}

	IEnumerator IEHttpGet()
	{
		//print ("get url " + fullUrl);
		WWW www = new WWW("http://127.0.0.1:6789/bmi");
		yield return www;
		
		if (www.error != null) {
			Debug.LogError (www.error);
		} 
		
		Debug.Log (www.text);
	}
}
```

# 3 简单的Post请求
这份代码使用的是JSON格式的数据。POST请求需要一个API地址，请求的HTTP头部需要加入头信息表明是json格式的数据，发送的josn数据格式化为字符串并转化为字节数据。这里使用的JSON解析库是MiniJSON。

```
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
using MiniJSON;

public class NewBehaviourScript : MonoBehaviour {

	// Use this for initialization
	void Start () {
		StartCoroutine (IEPost());
	
	}

	IEnumerator IEPost()
	{
		string fullUrl = "http://127.0.0.1:6789/bmi";

		//加入http 头信息
		Dictionary<string,string> JsonDic = new Dictionary<string,string> ();  
		JsonDic.Add ("Content-Type", "application/json");

		//请求的Json数据
		Dictionary<string,string> UserDic = new Dictionary<string, string> ();
		UserDic ["height"] = "170";
		UserDic ["weight"] = "62";
		string data = Json.Serialize (UserDic);

		//转换为字节
		byte[] post_data;  
		post_data = System.Text.UTF8Encoding.UTF8.GetBytes (data);  
		
		WWW www = new WWW (fullUrl, post_data, JsonDic);
		yield return www;
		
		if (www.error != null) {
			Debug.LogError ("error:" + www.error);
		} 
		Debug.Log (www.text);
	}
}

```

# 4 带安全验证的Get/Post请求

对于需要使用Base Auth 安全验证的RESTful API,需要在http的请求信息中加入用户名称和用户帐号信息，并进行编码转化。

新建一个字典数据，加入编码说明和帐号信息。

```
Dictionary<string,string> AuthDic = new Dictionary<string,string> ();  // auth header
AuthDic.Add("Content-Type", "application/json");
string NameAndPw = "UserName" + ":" + "Password";
AuthDic.Add("Authorization", "Basic " + System.Convert.ToBase64String(System.Text.Encoding.ASCII.GetBytes(NameAndPw)) );

```

完整的安全验证的POST请求

```
IEnumerator IEPost()
	{
		string fullUrl = "http://127.0.0.1:6789/bmi";



		Dictionary<string,string> AuthDic = new Dictionary<string,string> ();  // auth header
		AuthDic.Add("Content-Type", "application/json");
		string NameAndPw = "UserName" + ":" + "Password";
		AuthDic.Add("Authorization", "Basic " + System.Convert.ToBase64String(System.Text.Encoding.ASCII.GetBytes(NameAndPw)) );

		//请求的Json数据
		Dictionary<string,string> UserDic = new Dictionary<string, string> ();
		UserDic ["height"] = "170";
		UserDic ["weight"] = "62";
		string data = Json.Serialize (UserDic);

		//转换为字节
		byte[] post_data;  
		post_data = System.Text.UTF8Encoding.UTF8.GetBytes (data);  
		
		WWW www = new WWW (fullUrl, post_data, AuthDic);
		yield return www;
		
		if (www.error != null) {
			Debug.LogError ("error:" + www.error);
		} 
		Debug.Log (www.text);
	}
```

# 5 封装成带回调函数的请求

对于多次的http请求，如果编写多份协程来处理会显得计较麻烦。这里把HTTP的请求封装成一个回调函数。得到服务器返回的数据则马上调用回调函数进行处理。

以下的http请求的的封装。

```
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
using System;


/// <summary>
/// Http Request SDK 
/// </summary>
namespace RestHttpAPI
{
public class HttpAPI : MonoBehaviour {
    
		private static HttpAPI _instacne=null;
		private HttpAPI()
		{
		}

		public static HttpAPI Instance{
			get{
				if(_instacne == null )
				{
					Debug.LogError("Awake error");
				}
				return _instacne ;
			}
		}

		void Awake () {
			DontDestroyOnLoad (gameObject);
			HttpAPI._instacne = gameObject.GetComponent<HttpAPI> ();   
				
			JsonDic.Add("Content-Type", "application/json");

			AuthDic.Add("Content-Type", "application/json");
			string NameAndPw = UserName + ":" + Password;
			AuthDic.Add("Authorization", "Basic " + System.Convert.ToBase64String(System.Text.Encoding.ASCII.GetBytes(NameAndPw)) );

		}

		public string ipAdr="https://127.0.0.1:5000";
		public string ver = "";
		Dictionary<string,string> AuthDic = new Dictionary<string,string> ();  // auth header
		Dictionary<string,string> JsonDic = new Dictionary<string,string> ();  // json parser header

		public string UserName="";
		public string Password="";


		/// <summary>
		/// Get Method with callback
		/// </summary>
		/// <param name="url">URL.</param>
		/// <param name="callBack">Call back.</param>
		public void Get(string url,Action<string,string> callBack=null)
		{
			StartCoroutine ( IEHttpGet(this.ipAdr+url,callBack));
		}
		IEnumerator IEHttpGet(string fullUrl,Action<string,string> callBack)
		{
			//print ("get url " + fullUrl);
			WWW www = new WWW(fullUrl);
			yield return www;
			
			if (www.error != null) {
				Debug.LogError (www.error);
			} 

			if(callBack != null)
			   callBack (www.text,www.error);
		}

		/// <summary>
		/// Get Method with callback and  BaseAuth
		/// </summary>
		/// <param name="url">URL.</param>
		/// <param name="callBack">Call back.</param>
		public void GetAuth(string url,Action<string,string> callBack=null)
		{
			StartCoroutine (IEGetAuth(url,callBack));
		}
		IEnumerator IEGetAuth(string url,Action<string,string> callBack=null)
		{

			string fullUrl = this.ipAdr + url;


			WWW www = new WWW(fullUrl,null,AuthDic);
			yield return www;
			
			if (www.error != null) {
				Debug.LogError ("error:"+www.error);
			} 
			
			if(callBack != null)
				callBack (www.text,www.error);
		}

		/// <summary>
		/// Post the specified url and callBack.
		/// </summary>
		/// <param name="url">URL.</param>
		/// <param name="callBack">Call back.</param>
		public void Post(string url,Action<string,string> callBack=null)
		{
			StartCoroutine (IEPost (url, callBack));
		}
		IEnumerator IEPost(string url,Action<string,string> callBack=null)
		{
			string fullUrl = this.ipAdr + url;

			WWWForm wf = new WWWForm ();
			wf.AddField ("K","v");
			
			WWW www = new WWW(fullUrl,wf);
			yield return www;
			
			if (www.error != null) {
				Debug.LogError ("error:"+www.error);
			}
			
			if(callBack != null)
				callBack (www.text,www.error);
		}

		/// <summary>
		/// Post the specified url, data and callBack.
		/// </summary>
		/// <param name="url">URL.</param>
		/// <param name="data">Data.</param>
		/// <param name="callBack">Call back.</param>
		public void Post(string url,string data, Action<string,string> callBack=null)
		{
			StartCoroutine (IEPost(url,data,callBack));
		}
		IEnumerator IEPost(string url,string data, Action<string,string> callBack=null)
		{
			string fullUrl = this.ipAdr + url;
			
			byte[] post_data;  
			post_data = System.Text.UTF8Encoding.UTF8.GetBytes(data);  
			
			WWW www = new WWW(fullUrl,post_data,JsonDic);
			yield return www;
			
			if (www.error != null) {
				Debug.LogError ("error:"+www.error);
			} 
			if(callBack != null)
				callBack (www.text,www.error);
		}


		/// <summary>
		/// Posts the auth.
		/// </summary>
		/// <param name="url">URL.</param>
		/// <param name="data">Data.</param>
		/// <param name="callBack">Call back.</param>
		public void PostAuth(string url,string data, Action<string,string> callBack=null)
		{
			StartCoroutine (IEPostAuth(url,data,callBack));
		}
		IEnumerator IEPostAuth(string url,string data, Action<string,string> callBack=null)
		{
			string fullUrl = this.ipAdr + url;
			
			byte[] post_data;  
			post_data = System.Text.UTF8Encoding.UTF8.GetBytes(data);  
			
			WWW www = new WWW(fullUrl,post_data,AuthDic);
			yield return www;
			
			if (www.error != null) {
				Debug.LogError ("error:"+www.error);
			}
			
			if(callBack != null)
				callBack (www.text,www.error);
		}
	
}

}

```

# 6 使用示例

基于以上的RestHttpAPI可以使用简单的代码调用RESTful API。

```
//设置API的网络地址
HttpAPI.Instance.ipAdr = "https://127.0.0.1:5000";
HttpAPI.Instance.UserName = "b";
HttpAPI.Instance.Password = "123";

//普通的Get POST 请求
HttpAPI.Instance.Get ("/hello",(resp,error)=>{print(resp+" "+error);});
HttpAPI.Instance.Post ("/hello",(resp,error)=>{print(resp+" "+error);});

//带安全验证的GET请求
HttpAPI.Instance.GetAuth ("/test",(resp,error)=>{print(resp+" "+error);});

//POST请求
Dictionary<string,string> UserDic = new Dictionary<string, string> ();
UserDic["username"]="you";
UserDic["password"]="123";
string str = Json.Serialize(UserDic);
HttpAPI.Instance.Post ("/user/newUser", str, (resp,error) => {
	print (resp + " " + error);}
);

//带安装验证的POST 请求
Dictionary<string,string> UserDic = new Dictionary<string, string> ();
UserDic["appname"]="aote2";
UserDic["channel"]="haha";
UserDic["count"]="180";
string str = Json.Serialize(UserDic);
HttpAPI.Instance.PostAuth("/generateCode", str, (resp,error) => {
	print (resp + " " + error);}
);

```

# 7 参考资料

1. <http://www.ruanyifeng.com/blog/2014/05/restful_api.html>
2. <https://docs.unity3d.com/ScriptReference/WWW.html>
  
 