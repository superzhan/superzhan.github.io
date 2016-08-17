---
layout: post
title:  "unity surface shader 表面着色器"
date:   2016-08-16 14:01:54 +0800
categories: Blog
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - shader
---

# unity surface shader 表面着色器


### surface shader 结构

```
Shader "Surface_Example/DiffuseSimple" {
	SubShader {
		Tags { "RenderType"="Opaque" }
		LOD 200
		
		CGPROGRAM
	
		#pragma surface surf Lambert

		struct Input {
			float4 color : COLOR;
		};


		void surf (Input IN, inout SurfaceOutput  o) {
		    o.Albedo=0.5f;
		}
		ENDCG
	} 
	FallBack "Diffuse"
}
```

### 输入、输出结构体

定义输入结构体

uv_+属性名称 表示这是一个贴图的UV坐标。
```
struct Input {
  float2 uv_MainTex;
  float3 worldNormal;
  float3 viewDir;
};
```

输入结构中还有一些固定作用的变量。

```
float3 viewDir - will contain view direction, for computing Parallax effects, rim lighting etc.

float4 with COLOR semantic - will contain interpolated per-vertex color.

float4 screenPos - will contain screen space position for reflection or screenspace effects.

float3 worldPos - will contain world space position.

float3 worldRefl - will contain world reflection vector if surface shader does not write to o.Normal. See Reflect-Diffuse shader for example.

float3 worldNormal - will contain world normal vector if surface shader does not write to o.Normal.

float3 worldRefl; INTERNAL_DATA - will contain world reflection vector if surface shader writes to o.Normal. To get the reflection vector based on per-pixel normal map, use WorldReflectionVector (IN, o.Normal). See Reflect-Bumped shader for example.

float3 worldNormal; INTERNAL_DATA - will contain world normal vector if surface shader writes to o.Normal. To get the normal vector based on per-pixel normal map, use WorldNormalVector (IN, o.Normal).
```



Lambert 光照模型的输出结构体

```
struct SurfaceOutput
{
    fixed3 Albedo;  // diffuse color
    fixed3 Normal;  // tangent space normal, if written
    fixed3 Emission;
    half Specular;  // specular power in 0..1 range
    fixed Gloss;    // specular intensity
    fixed Alpha;    // alpha for transparencies
};
```

unity 5 物理光照的输出结构体

```
struct SurfaceOutputStandard
{
    fixed3 Albedo;      // base (diffuse or specular) color
    fixed3 Normal;      // tangent space normal, if written
    half3 Emission;
    half Metallic;      // 0=non-metal, 1=metal
    half Smoothness;    // 0=rough, 1=smooth
    half Occlusion;     // occlusion (default 1)
    fixed Alpha;        // alpha for transparencies
};

//镜面反射
struct SurfaceOutputStandardSpecular
{
    fixed3 Albedo;      // diffuse color
    fixed3 Specular;    // specular color
    fixed3 Normal;      // tangent space normal, if written
    half3 Emission;
    half Smoothness;    // 0=rough, 1=smooth
    half Occlusion;     // occlusion (default 1)
    fixed Alpha;        // alpha for transparencies
};
```

### #pragma 编译指令

pragma 用来指明处理像素的函数名称，处理定点的函数名称。
```
#pragma surface surf Lambert vertex:vert
```
指明了surf函数 和光照模型，vertex 指明处理顶点的函数。


### Example

#### 简单描边

```
Shader "Surface_Example/Rim" {
	
	 Properties {
    _MainTex ("Albedo (RGB)", 2D) = "white" {}
    _Color ("Color", Color) = (1, 1, 1, 1)
    _Rim ("Rim", Range(0,1)) = 0.1
  }
  SubShader {
    Tags { "RenderType" = "Opaque" }
    CGPROGRAM
    #pragma surface surf Standard

    sampler2D _MainTex;
    float4 _Color;
    float _Rim;
    #define RIM (1.0 - _Rim)

    struct Input {
        float2 uv_MainTex;
        float3 worldNormal;
        float3 viewDir;
    };

    void surf (Input IN, inout SurfaceOutputStandard o) {

      // Calculate the angle between normal and view direction
      float diff = 1.0 - dot(IN.worldNormal, IN.viewDir);

      // Cut off the diff to the rim size using the RIM value.
      diff = step(RIM, diff) * diff;

      // Smooth value
      float value = step(RIM, diff) * (diff - RIM) / RIM;

      // Sample texture and add rim color
      float3 rgb = tex2D(_MainTex, IN.uv_MainTex).rgb;
      o.Albedo = value * _Color + rgb;
    }
    ENDCG
  }
  FallBack "Diffuse"
}
```

#### 外发光

```
Shader "Surface_Example/EdgeMultiPass" {
	
	 Properties {
    _MainTex ("Albedo (RGB)", 2D) = "white" {}
    _OutlineSize ("Outline Size", float) = 0.05
    _OutlineColor ("Outline Color", Color) = (1, 1, 1, 1)
  }
  SubShader {
     Tags { "RenderType" = "Opaque" }

    // Render the normal content as a second pass overlay.
    // This is a normal texture map.
    CGPROGRAM
      #pragma surface surf Standard

      sampler2D _MainTex;

      struct Input {
        float2 uv_MainTex;
      };

      void surf (Input IN, inout SurfaceOutputStandard o) {
        fixed4 c = tex2D (_MainTex, IN.uv_MainTex);
        o.Albedo = c.rgb;
      }
    ENDCG

    // Cull rendered pixels on this surface
    Cull Front

    // Render scaled background geometry
    CGPROGRAM
      #pragma surface surf Standard vertex:vert

      float4 _OutlineColor;
      float _OutlineSize;

      // Linearly expand using surface normal
      void vert (inout appdata_full v) {
        v.vertex.xyz += v.normal * _OutlineSize;
      }

      struct Input {
        float2 uv_MainTex;
      };

      void surf (Input IN, inout SurfaceOutputStandard o) {
        o.Albedo = _OutlineColor.rgb;
      }
    ENDCG
  }
  FallBack "Diffuse"
}
```

#### 积雪

![img](/img/post/Snow.png) 

```
hader "Custom/SnowShader" {
	Properties {
		_MainColor("Main Color",Color) = (1,1,1,1)
		_MainTex("Main texture",2D)="white"{}
		_Bump("Bump tex",2D)="white"{}
		_Snow("Level of snow",Range(1,-1))=1
		_SnowColor("Color of snow",Color)=(1,1,1,1)
		_SnowDirection("Direction of snow",Vector)=(0,1,0)
		_SnowDepth("Depth of snow",Range(0,0.0001)) =0
	}
	SubShader {
		
		Tags{"RenderType"="Opaque"}
		LOD 200
		
		CGPROGRAM
		#pragma surface surf Lambert vertex:vert
		
		sampler2D _MainTex;
		sampler2D _Bump;
		float _Snow;
		float4 _SnowColor;
		float4 _MainColor;
		float4 _SnowDirection;
		float _SnowDepth;
		
		
		struct Input{
		    float2 uv_MainTex;
		    float2 uv_Bump;
		    float3 worldNormal; INTERNAL_DATA
		};
		
		void vert(inout appdata_full v)
		{
                    //把_SnowDirection 变为世界坐标的方向
		    float4 sn=mul( transpose(_Object2World), _SnowDirection );
		    if( dot( v.normal ,sn.xyz) >=_Snow )
		    {
		        v.vertex.xyz += (sn.xyz + v.normal)*_SnowDepth*_Snow;
		    }
		}
		
		
		void surf(Input IN,inout SurfaceOutput o)
		{
		    half4 c = tex2D(_MainTex,IN.uv_MainTex);
		    o.Normal = UnpackNormal( tex2D(_Bump,IN.uv_Bump) );
		    
                     //和 雪的方向相同 就 渲染为白色
		    if( dot( WorldNormalVector(IN,o.Normal) ,_SnowDirection.xyz )>=_Snow )
		    {
		        o.Albedo = _SnowColor.rgb;
		    }else
		    {
		        o.Albedo = c.rgb*_MainColor;
		    }
		    o.Alpha=1;
		}
		ENDCG
		
	} 
	FallBack "Diffuse"
}
```
