---
layout: post
title: "AssetBundle的简单使用"
date: 2017-09-06
description: "基于Unity5.5的AssetBundle(AB包)的简单使用方法"
tag: 博客 
---   

最近看到了一个很好的游戏开发框架GameFream，正在学习中，它封装了一套的AB包的打包工具以及管理化的更新下载，感觉非常好用，充一下电，查阅了一些网上的博客以及翻阅5.X书籍整理了一份AssetBundle的简单使用。


# 概述
平时我们讲资源放入Unity项目的目录内时编辑器会自动帮我们将资源编码成Unity能识别的资源（会生成.meta文件）以便项目加载使用。`Resources.Load()`是我们常见的资源加载方式，但仅限于对`Assets/Resources`目录下的资源加载，并且此目录在打包时会被全部打到包里去，这种方式适用于在开发项目起初就定死的一些资源。而有些项目需要热更新替换部分原资源以及动态新增加一些资源，这对于已经发布的项目来说这种加载方式就不能满足了。  

此时，Unity的另一种资源加载方式就能很好的解决这个问题--`AssetBundle`，俗称AB包。它能将资源文件通过Unity的API或者工具打包成一个AB资源包文件，然后在代码中通过API读取这个资源包来加载包里面的资源文件。它的好处在于它完全支持动态更改，不用在项目起初就决定，项目只需要更新AB包而不需要从新下载项目包就能达到更新效果。  

# 制作AssetBundle资源包
## 准备资源
在Unity中找到需要打包的资源（图片、文本、预设等等要打包的资源），选中它会在Unity的属性面板右下角看到AssetBundle属性设置。  
![AssetBundle属性设置](/images/posts/posts-20170906/01.jpg)  
图中①②分别为打包出来后的主名和后缀名，设置好后打包出来将输出该文件名的AB包文件。几个文件都设为相同名称的话，将会把它们都打包到该AB包中。  
这里值得一提的是所有输出的AB包在打包时都会生成对应的`Manifest`文件，它是用来处理AB包内资源的依赖关系。假设这个AB包中的A、B预设分别用到了J图片，打包过程中不会重复将J图打入包中，如果A做了改变不用J图了，系统打包也不会全部重新打包，只更新A到包中。

## 打包
打AB资源包需要用到API：`BuildPipeline.BuildAssetBundles`，为了方便使用，制作一个编辑器运行脚本来执行打包。新建一个AssetBundles文件夹作为AB包的输出目录。  
```csharp
[MenuItem("Assets/Build AssetBundles")]
    static void BuildAllAssetBundles()
    { 
        BuildPipeline.BuildAssetBundles(Application.dataPath + "/AssetBundles",
         BuildAssetBundleOptions.ChunkBasedCompression, BuildTarget.StandaloneWindows);
    }
```
此API的参数第一个为输出路径，第二个为打包设置(压缩、带hash、完整、全部重新打包..等等)，第三个为目标平台。
将此脚本放入Editor文件夹编译成功后，编辑器的Assets菜单下增加了BuildAllAssetBundles的选项，单击过后就开始打AB包了。  
![AB包输出目录](/images/posts/posts-20170906/02.jpg)  
打包后增加了四个文件，分别对应了总包的AB文件和对应的Manifest文件，以及测试用的unlock.bb的AB包及Manifest文件。  

# 加载AssetBundle资源包

## 加载AssetBundle  
非缓存机制的加载，下载后不会放入Unity引擎特定缓存区。 (url为本地或远端的AB包路径)
>WWW www = new WWW(url);  
AssetBundle assetBundle = www.assetBundle;

此模式下也可以通过`AssetBundle.LoadFromMemoryAsync(www.bytes);`来获取AssetBundle对象，这里有个好处就是通过字节来创建的AssetBundle对象，这意味着可以先解密www.bytes，再通过此方式来创建AssetBundle。

缓存机制加载（推荐），加载后会自动放入到缓存区内，如果该文件在之前已经被载入缓存区，则直接从缓存区读取。（参数二为版本号）
>WWW www = WWW.LoadfromCacheorDownload(url,0);  
AssetBundle assetBundle = www.assetBundle;

## 从AssetBundle中加载Assets资源

```csharp
GameObject bb = assetBundle.LoadAsset<GameObject>("Assets/Art/UI_new/Prefab/Canvas_Unlock.prefab");
GameObject.Instantiate(bb);
```
AssetBundle的LoadAsset还有异步加载方式。

# 完整代码
```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class testAssetsBundles : MonoBehaviour
{
    //总manifest文件名
    string manifestName = "AssetBundles";
    //加载目标文件名
    string assetBundleName = "unlock.bb";

    void Start ()
    {
        StartCoroutine(LoadAssetBundle());
    }

    IEnumerator LoadAssetBundle()
    {
        //AB包输出路径
        string assetBundlePath = "file:///" + Application.dataPath + "/AssetBundles/";
        //总Manifest文件路径
        string manifestPath = assetBundlePath + manifestName;

        //首先加载Manifest文件
        WWW wwwManifest = WWW.LoadFromCacheOrDownload(manifestPath, 0);
        yield return wwwManifest;
        if (wwwManifest.error == null)
        {
            //解析主Manifest的依赖文件
            AssetBundle manifestBundle = wwwManifest.assetBundle;
            AssetBundleManifest manifest = (AssetBundleManifest)manifestBundle.LoadAsset("AssetBundleManifest");
            manifestBundle.Unload(false);
            //获取目标文件所依赖文件列表
            string[] dependentAssetBundles = manifest.GetAllDependencies(assetBundleName);
            AssetBundle[] abs = new AssetBundle[dependentAssetBundles.Length];
            for (int i = 0; i < dependentAssetBundles.Length; i++)
            {
                //加载所有依赖文件
                WWW www = WWW.LoadFromCacheOrDownload(assetBundlePath + dependentAssetBundles[i], 0);
                yield return www;
                abs[i] = www.assetBundle;
            }
            
            //缓存机制加载（推荐）先检查本地缓存区如果有下载过则直接从缓存区读入
            WWW www2 = WWW.LoadFromCacheOrDownload(assetBundlePath + assetBundleName, 2);
            //如虚解密数据，则用此方式加载，这种方法能够获取bytes数据
            //WWW www2 = new WWW(assetBundlePath + assetBundleName);
            yield return www2;
            if (www2.error != null)
            {
                Debug.Log(www2.error);
                yield return null;
            }
            //通常获取
            AssetBundle assetBundle = www2.assetBundle;

            //内存获取(对信息有解密操作后可用次方式)
            //AssetBundleCreateRequest assetBundleCreateRequest = AssetBundle.LoadFromMemoryAsync(www2.bytes);
            //yield return assetBundleCreateRequest;
            //AssetBundle assetBundle = assetBundleCreateRequest.assetBundle;

            //从AssetBundle中获取资源
            //AssetBundleRequest request = assetBundle.LoadAssetAsync<GameObject>("Assets/Art/UI_new/Prefab/Canvas_Unlock.prefab");
            //yield return request;

            //GameObject bb = GameObject.Instantiate(request.asset as GameObject);

            GameObject bb = assetBundle.LoadAsset<GameObject>("Assets/Art/UI_new/Prefab/Canvas_Unlock.prefab");
            GameObject.Instantiate(bb);
        }
        else
        {
            Debug.Log(wwwManifest.error);
        }
    }
}
```

