---
layout: post
title:  "Unity5 AssetBundle的简单使用"
date:   2017-2-19 15:31:54 +0800
categories: Blog
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    - Unity
---

# 1 AssetBundle

AssetBundle 是Unity提供的一个功能，可以把资源（包括预设、模型、贴图等）压缩成一个资源包，可以应到到游戏的热更新上。
  

# 2 AssetBundel 打包

Unity5 简化了AssetBundle的打包步骤，只需要给每一个需要打包的资源定义报名，然后通过BuildPipeline的API，就可以导出一个资源包。

![img](/img/2017-2-19-Unity5AssetBundle/项目图.png)

![img](/img/2017-2-19-Unity5AssetBundle/Cube.png)

Step1 新建一个Cube对象，把它制作成预设，在资源预览窗口中设置Cube的名称和变量。参考上图，ab-cube 是 Cube 对象的名称，a 是资源名变量。 最后通过 ab-cube.a 的定位这个资源包。

Step2 使用编辑器脚本打包资源。在项目中新建一个Editor的文件夹，新建一个脚本AssetBundleBuilder。这个脚本在工程中创建了一个目录，然后把做了AssetBundle 打包标记的的资源进行打包处理。执行菜单-> Assets -> BuildAssetBundle ,Unity 就会开始进行打包进程。

``` C#
using UnityEngine;
using System.Collections;
using UnityEditor;
using System.IO;
public class AssetBundleBuilder 
{
	[MenuItem("Assets/Build AssetBundle")]
	static public void BuildAssetBundle()
	{
		Caching.CleanCache ();
		string path = Application.streamingAssetsPath + "/" + "AssetBundles" + "/" + "OSX";
		if (!Directory.Exists (path)) 
		{
			Directory.CreateDirectory(path);
		}
		BuildPipeline.BuildAssetBundles (path,0,EditorUserBuildSettings.activeBuildTarget);
		AssetDatabase.Refresh ();
	}
}

```

打包进程结束后的AssetBundle 文件。结果有4个文件，其中两个是.manifest 文件，两个是资源包文件。 manifest 包含了资源的依赖项和资源的打包说明，OSX是总包，ab-cube.a 才是包含Cube 的资源包。
![img](/img/2017-2-19-Unity5AssetBundle/打包结果.png)


# 3 从本地加载AssetBundle

开发的时候可以先从本地加载AssetBundle,真正发行游戏的时候才做网络热更新加载，这样会比较方便。

新建一个脚本,把脚本绑定在一个GameObject 上，点击运行。 Cube 预设就会从AssetBundle 中加载出来。

``` C#
using UnityEngine;
using System.Collections;
using System.Collections.Generic;

public class LoadAssetBundle : MonoBehaviour {

	public string url;
	public int version;
	public string assetName;
	public string assetBundleName;

	string manifestName;

	void Start () 
	{
		InitializeAssetBundlePath ();
		StartCoroutine(LoadAssetBundleAsObject());
	}

	void InitializeAssetBundlePath()
	{
		#if UNITY_EDITOR

		url = "file://" + Application.streamingAssetsPath+"/AssetBundles/OSX/";
		assetBundleName ="ab-cube.a"; 
		assetName="Cube";
		#endif
	}

	public IEnumerator LoadAssetBundleAsObject()
	{

		WWW www2 = new WWW (url + assetBundleName);
		yield return www2;

		if (!string.IsNullOrEmpty (www2.error)) {
			Debug.Log (www2.error);
		} else {
			AssetBundle ab = www2.assetBundle;
			GameObject gobj = ab.LoadAsset (assetName) as GameObject;
			Instantiate (gobj);
		}
	
		www2.Dispose ();
	}


}


```

# 4 从网络中加载AssetBundle

Unity的资源热更新需要把AssetBundle的资源包上传到网络，然后再通过WWW的接口下载到本地。这里使用的Mac系统自带的apache,可以参考[Mac OS X 上的Apache配置](http://www.jianshu.com/p/7b8d5d6f22c9) 。

step1 把资源包ab-cube.a （只需要这一个文件） 放到apache的根目录中，可以在浏览器中通过网址 http://127.0.0.1/ab-cube.a 来访问这个资源包。

step2 新建一个脚本并绑定在场景中。

```
using UnityEngine;
using System.Collections;

public class WebLoadAsset : MonoBehaviour {

	// Use this for initialization
	void Start () {
		StartCoroutine (LoadAssets());
	}
	IEnumerator LoadAssets()
	{
		WWW w = WWW.LoadFromCacheOrDownload ("http://127.0.0.1/ab-cube.a",0);
		yield return w;
		AssetBundle ab = w.assetBundle;
		GameObject go= ab.LoadAsset<GameObject> ("Cube");
		Instantiate (go);
	}
}

```

# PS

1 不同平台的AssetBundle包是不兼容的，android的资源包只能用于android平台，ios的资源包只能用在ios平台。

2 windows和mac 加载资源包时，文件路径前需要添加 "file://"。例如 url = "file://" + Application.streamingAssetsPath+"/AssetBundles/OSX/";

3 要注意资源的依赖关系，公用的资源尽量独立处理，以免重复打包。

4 以上是简单示例，没有做错误处理。

# 5 参考资料

1 [Unity3D研究院之Assetbundle的原理（六十一）](http://www.xuanyusong.com/archives/2373)

2 [AssetBundles 官方文档](https://docs.unity3d.com/Manual/AssetBundlesIntro.html)