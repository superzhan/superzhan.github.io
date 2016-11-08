---
layout: post
title:  "unity 曲线编辑 曲线移动 路径运动"
date:   2016-08-25 14:01:54 +0800
categories: Blog
header-img: "img/tag-bg.jpg"
catalog: true
tags:
    - Unity
---

### 获取平滑的曲线

 在开发按照路径移动的的游戏中，需要编辑一条特殊的路径。 这通常在这条路径上摆放几个坐标点，然后根据这些点生成一条平滑的曲线。
 通过N个顶点使用插值算法可以得到一条平滑的曲线，这里使用样条曲线。
 
```
public static List<Vector3> GetSmoothPath(List<Vector3> pathList)
{
	List<Vector3> smoothPath=new List<Vector3>();
	if(pathList==null || pathList.Count<2)
	{
		return smoothPath;
	}

	Vector3[] path = pathList.ToArray();
	Vector3[] vector3s = PathControlPointGenerator(path);
     
      //控制平滑度
	int SmoothAmount = path.Length*100;
	for (int i = 0; i <= SmoothAmount; i++) {
		float pm = (float) i / SmoothAmount;
		Vector3 currPt = Interp(vector3s,pm);
		smoothPath.Add(currPt);
	}
	return smoothPath;
}
	
private static Vector3 Interp(Vector3[] pts, float t)
{
	int numSections = pts.Length - 3;
	int currPt = Mathf.Min(Mathf.FloorToInt(t * (float) numSections), numSections - 1);
	float u = t * (float) numSections - (float) currPt;
	
	Vector3 a = pts[currPt];
	Vector3 b = pts[currPt + 1];
	Vector3 c = pts[currPt + 2];
	Vector3 d = pts[currPt + 3];
	
	return .5f * (
		(-a + 3f * b - 3f * c + d) * (u * u * u)
		+ (2f * a - 5f * b + 4f * c - d) * (u * u)
		+ (-a + c) * u
		+ 2f * b
		);
}	
	
private static Vector3[] PathControlPointGenerator(Vector3[] path)
{
	Vector3[] suppliedPath;
	Vector3[] vector3s;
	
	//create and store path points:
	suppliedPath = path;
	
	//populate calculate path;
	int offset = 2;
	vector3s = new Vector3[suppliedPath.Length+offset];
	Array.Copy(suppliedPath,0,vector3s,1,suppliedPath.Length);
	
	//populate start and end control points:
	//vector3s[0] = vector3s[1] - vector3s[2];
	vector3s[0] = vector3s[1] + (vector3s[1] - vector3s[2]);
	vector3s[vector3s.Length-1] = vector3s[vector3s.Length-2] + (vector3s[vector3s.Length-2] - vector3s[vector3s.Length-3]);
	
	//is this a closed, continuous loop? yes? well then so let's make a continuous Catmull-Rom spline!
	if(vector3s[1] == vector3s[vector3s.Length-2]){
		Vector3[] tmpLoopSpline = new Vector3[vector3s.Length];
		Array.Copy(vector3s,tmpLoopSpline,vector3s.Length);
		tmpLoopSpline[0]=tmpLoopSpline[tmpLoopSpline.Length-3];
		tmpLoopSpline[tmpLoopSpline.Length-1]=tmpLoopSpline[2];
		vector3s=new Vector3[tmpLoopSpline.Length];
		Array.Copy(tmpLoopSpline,vector3s,tmpLoopSpline.Length);
	}	
	
	return(vector3s);
}
```
 
 使用GetSmoothPath 函数，传入N个顶点坐标的List 作为参数，最后得到平滑曲线的顶点List。 曲线的平滑度由SmoothAmount决定，数字越大，越平滑。
 
 Interp 函数是样条曲线的插值函数，根据参数t 获取该位置的顶点。
 
 PathControlPointGenerator 函数做了路径的循环处理。
 
 ![img](/img/post/unity_spline.png)
 
### 计算曲线长度 计算曲线上的顶点
 
 接下来计算曲线的长度CalPathLen,用一个列表纪录每一个点到开始点的距离。
 
 使用折半查找，根据输入的长度（距离开始点的长度）计算距离开始点长度为len 时的顶点位置。
 
```

private void CalPathLen()
{
	if(smoothPointList == null || smoothPointList.Count<2)
	{
		Debug.Log("no path");
		return;
	}
	pointLenInPath.Clear();
	pointLenInPath.Add(0);
	pathLen=0;
	for(int i=1;i<smoothPointList.Count;++i)
	{
		float d = Vector3.Distance(smoothPointList[i-1],smoothPointList[i]);
		pathLen+=d;
		pointLenInPath.Add(pathLen);
	}
}

public Vector3 GetPointByLen(float len)
{
	if(len> pathLen)
		len=len%pathLen;

	int startIndex=0;
	int endIndex=smoothPointList.Count-1;
	int middleIndex= (startIndex+endIndex)/2;

	while(endIndex - startIndex >1)
	{
		if(len>=pointLenInPath[middleIndex])
		{
			startIndex=middleIndex;
			middleIndex=  (startIndex+endIndex)/2;
		}else
		{
			endIndex= middleIndex;
			middleIndex= (startIndex+endIndex)/2;
		}
	}

	float delLen=len - pointLenInPath[startIndex];
	float delPercent = delLen / ( pointLenInPath[startIndex+1] - pointLenInPath[startIndex] );

	Vector3 pos = Vector3.Lerp(smoothPointList[startIndex] ,smoothPointList[startIndex+1], delPercent);

	
	return pos;
}
```

### 按照平滑曲线移动

   计算后路径之后，移动的代码就相对简单了。
   
   在updata() 中调用 MoveToNextPoint(),每一帧移动一次，并设置移动的朝向。

```
void MoveToNextPoint()
{
	moveLen+=speed * Time.deltaTime;
	selfTrans.position = roadPoint.GetPointByLen(moveLen,xOffset);
	nextPos = roadPoint.GetPointByLen(moveLen + 1f, xOffset);
	selfTrans.LookAt(nextPos);

}
```


