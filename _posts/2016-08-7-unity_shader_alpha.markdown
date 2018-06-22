---
layout: post
title:  "unity shader 透明设置 alpha 和 cutoff alpha"
date:   2016-08-7 14:01:54 +0800
categories: Blog
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    - shader
---

# unity shader 透明设置 alpha 和 cutoff alpha


### 设置3D模型透明 alpha

###### 目录

	1. 普通透明设置准备
	2. 普通透明设置实现
	3. 普通透明设置说明
	
	4. cutoff准备
	5. cutoff实现
	6. cutoff说明

Unity默认使用的shader不能设置3D对象的透明和和半透明，我们只需要做简单的修改就可以实现这个功能。

###### 普通透明设置准备

1. 新建一个 Stander surface shader 和一个材质material,命名为NormalAlpha，把新建的shader绑定在这个材质上。
2. 准备一个简单的模型和对应的贴图，这里是从asset store下载了一个卡通车模型。
3. 把新建的material 赋给卡通车模型。

##### 普通透明设置实现

1. 添加渲染队列标签，把这个标签添加到tags{}中。

    `"Queue"="Transparent"`
2. 在#pragma编译指令后添加参数 alpha

    ```
	#pragma surface surf Lambert alpha 
	```
3. 使用颜色值的最后一位设置颜色值，附上surf函数。

```c
void surf (Input IN, inout SurfaceOutput o) {
	// Albedo comes from a texture tinted by color
	fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
	o.Albedo = c.rgb;
	o.Alpha = c.a;
}
``` 
   可以通过属相面板的颜色值来控制透明度，因为`o.Alpha = c.a;`。也可以直接设定透明度，比如`o.Alpha = 0.5; `。
   
   **完整代码**
   
   代码已经上传到github,[https://github.com/superzhan/shaderProj](https://github.com/superzhan/shaderProj)。
透明控制的代码在Alpha文件夹下。
   
```c
Shader "Custom/Alpha/NormalAlpha" {
	Properties {
		_Color ("Color", Color) = (1,1,1,1)
		_MainTex ("Albedo (RGB)", 2D) = "white" {}
		
	}
	SubShader {
		Tags { "RenderType"="Opaque" "Queue"="Transparent" }
		LOD 200
		
		CGPROGRAM
		
		//再最后添加alpha参数，就可以实现模型的透明设置
		#pragma surface surf Lambert alpha 

		// Use shader model 3.0 target, to get nicer looking lighting
		#pragma target 3.0

		sampler2D _MainTex;

		struct Input {
			float2 uv_MainTex;
		};

		
		fixed4 _Color;

		void surf (Input IN, inout SurfaceOutput o) {
			// Albedo comes from a texture tinted by color
			fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
			o.Albedo = c.rgb;
			o.Alpha = c.a;
		}
		ENDCG
	} 
	FallBack "Diffuse"
}

```


##### 普通透明设置说明
透明的设置很简单，主要是添加渲染队列的tag和在编译指令后添加alpha参数，最后在surf函数中设置输出的SurfaceOutput的alpha值。这里本来使用unity5 的默认光照模型，但设置透明的时候出来点问题，然后就改成了Lambert光照模型，surf函数的输出结构也改成了SurfaceOutput。



### cutoff alpha 通道透明设置

cutoff alpha的功能是当alpha值小于某个值的时候，就把这个像素点设置成完全透明。用来实现某些效果，比如腐烂消失等。

##### cutoff准备

1. 新建一个 Stander surface shader 和一个材质material,命名为CutoffAlpha，把新建的shader绑定在这个材质上。
2. 准备一个简单的模型和对应的贴图，这里是从asset store下载了一个卡通车模型。
3. 把新建的material 赋给卡通车模型。

##### cutoff实现

1. 添加渲染队列标签，把这个标签添加到tags{}中。

    `"Queue"="Transparent"`
2. 在Properties添加一个属性,用于控制alpha的范围，当alpha小于这个数值时，就会把像素点设置透明。 

   ```c
   _Cutoff("Cutoff Value",Range(0,1.1))=0.5
   ```
3. 在#pragma编译指令后添加参数 alphatest:_Cutoff 。 冒号后的_Cutoff就是属性中的数值。

    ```c
	#pragma surface surf Lambert alphatest:_Cutoff 
	```
4. 在surf函数中设置要变更的颜色通道。

```c
void surf (Input IN, inout SurfaceOutput o) {
	// Albedo comes from a texture tinted by color
	fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
	o.Albedo = c.rgb;
		
	//这里使用蓝色通道
	o.Alpha = c.b;
}
```

**附上完整代码**

代码已经上传到github,[https://github.com/superzhan/shaderProj](https://github.com/superzhan/shaderProj)。
透明控制的代码在Alpha文件夹下。

```c
Shader "Custom/Alpha/CutoffAlpha" {
	Properties {
		_Color ("Color", Color) = (1,1,1,1)
		_MainTex ("Albedo (RGB)", 2D) = "white" {}
		_Cutoff("Cutoff Value",Range(0,1.1))=0.5
		
	}
	SubShader {
		Tags { "RenderType"="Opaque" "Queue"="Transparent" }
		LOD 200
		
		CGPROGRAM
		
		#pragma surface surf Lambert alphatest:_Cutoff

		// Use shader model 3.0 target, to get nicer looking lighting
		#pragma target 3.0

		sampler2D _MainTex;

		struct Input {
			float2 uv_MainTex;
		};

		fixed4 _Color;

		void surf (Input IN, inout SurfaceOutput o) {
			// Albedo comes from a texture tinted by color
			fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
			o.Albedo = c.rgb;
			
			o.Alpha = c.b;
		}
		ENDCG
	} 
	FallBack "Diffuse"
}

```

![img](/img/2016-8-7-unityalpha/alpha1.png) ![img](/img/2016-8-7-unityalpha/alpha2.png) ![img](/img/2016-8-7-unityalpha/alpha3.png)

##### cutoff说明
当设置`alphatest:_Cutoff`后，`o.Alpha`输出结构的透明值小于_Cutoff，这个像素点就会完全透明。在surf函数中的`o.Alpha = c.b `可以设置为`o.Alpha = c.r `或者`o.Alpha = c.b `,不同的贴图颜色会有不同的效果，需要根据具体的情况做调整。


    
 