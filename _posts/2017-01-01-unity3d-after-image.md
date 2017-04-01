---
layout: single
categories: unity3d
---
多谢[Unity实现残影特效](http://blog.csdn.net/langresser_king/article/details/50976029)这篇博文，让我发现了unity3d居然提供了Graphics.DrawMesh这样强大的功能。基于Graphics.DrawMesh，完成了残影效果
核心core如下
```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
using System.Linq;

// 残影效果  
public class AfterImageComponent : MonoBehaviour
{
	class AfterImage
	{
		public Mesh mesh;
		public Material material;
		public float showStartTime;
		public float duration;  // 残影镜像存在时间  
		public float alpha;
		public bool needRemove = false;
		public Quaternion quat;
		public Vector3 pos;
	}

	private float _duration; // 残影特效持续时间  
	private float _interval; // 间隔  
	private float _fadeTime; // 淡出时间  

	private List<AfterImage> _imageList = new List<AfterImage>();
	private Shader _shaderAfterImage;

	void Awake()
	{
		_shaderAfterImage = Shader.Find("Muffin/MuffinPhantom");
	}

	public void Play(float duration, float interval, float fadeout)
	{
		_duration = duration;
		_interval = interval;
		_fadeTime = fadeout;

		StartCoroutine(DoAddImage());
	}

	IEnumerator DoAddImage()
	{
		float startTime = Time.realtimeSinceStartup;
		while (true)
		{
			CreateImage();

			if (Time.realtimeSinceStartup - startTime > _duration)
			{
				break;
			}

			yield return new WaitForSeconds(_interval);
		}
	}

	private void CreateImage()
	{
		// 获取skin mesh
		SkinnedMeshRenderer[] renderers = GetComponentsInChildren<SkinnedMeshRenderer>();

		Transform t = transform;
		Material mat = null;
		for (int i = 0; i < renderers.Length; ++i)
		{
			var item = renderers[i];
			var tK = item.transform;

			// 创建残影材质球
			//mat = new Material(_shaderAfterImage);
			//mat.SetTexture("_normalMap", item.material.GetTexture("_NormalTex"));
			mat = item.material;

			// bake skin mesh。 这里因为DrawMesh不能画出skin mesh。所以先bake成普通mesh
			var mesh = new Mesh();
			item.BakeMesh(mesh);

			// 这里处理动画旋转的问题。 因为用到了scale.x = -1 来旋转动画，这里做了旋转
			var count = mesh.vertexCount;
			var tmp = mesh.vertices;
			var baseScalex = tK.lossyScale.x;
			for (int j = 0; j < count; j++)
			{
				// 这里注意不要用mesh.vertices 要不效率会超低
				tmp[j].x *= baseScalex;
			}
			mesh.RecalculateBounds();
			mesh.RecalculateNormals();

			// 加入显示队列
			_imageList.Add(new AfterImage
			{
				mesh = mesh,
				material = mat,
				showStartTime = Time.realtimeSinceStartup,
				duration = _fadeTime,
				quat = tK.transform.rotation,
				pos = tK.transform.position
			});

		}
	}

	void LateUpdate()
	{
		bool hasRemove = false;
		foreach (var item in _imageList)
		{
			float time = Time.realtimeSinceStartup - item.showStartTime;

			if (time > item.duration)
			{
				item.needRemove = true;
				hasRemove = true;
				continue;
			}

			Graphics.DrawMesh(item.mesh, item.pos, item.quat, item.material, gameObject.layer);
		}

		if (hasRemove)
		{
			_imageList.RemoveAll(x => x.needRemove);
		}
	}
}
```
这里有个比价大的坑。因为模型加了scale.x=-1后，模型本身并不会旋转。**实在unity3d的动画计算的时候，把mesh翻转的**做的时候，这里浪费了不少时间。
[sourcecode](https://github.com/jesliwang/CommonTools/blob/master/Assets/Common/Script/AfterImageComponent.cs)
