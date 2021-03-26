https://docs.unity3d.com/Packages/com.unity.addressables@1.16/manual/index.html

#### AddressableAssetSettings
Addressable的设置文件
1. profile in use: 这个可以选择配置(一般需要有编辑器下、发布两种环境)
2. Catalog
   1. Disable Catalog Update on Startup:这个勾选了等于说要自己去写下载逻辑，手游一般要勾选
   2. Player Version Override:默认是catalog_2021.03.26.07.32.22类似这种，这里可以填打包机器生成的catalog，然后设置了remoteloadpath之后，就可以共享一个工程了
   3. Compress Local Catalog:压缩，一般不勾选
   4. Optimize Catalog Size:优化目录大小(加载目录时会额外占用cpu时间)
   5. Build Remote Catalog: 是否打服务器包，勾选了会出现以下BuildPath和LoadPath
      1. BuildPath
      2. LoadPath
3. General
   1. Send Profiler Events:勾选后在Event Viewer里面可以看到资源加载情况、引用情况
   2. Log Runtime Exceptions
   3. Custom certificate handler：继承与CertificateHandler，负责拒绝或接受在 https 请求上接收到的证书。当使用UnityWebRequest来实现HTTPS通信时，可能会使用 参考https://blog.csdn.net/weixin_45109984/article/details/103696944
   4. Unique Bundles IDs：会生成更复杂的内部名称的资源包。可能会导致重建时生成更多的资源包，但对于更新可能比较安全
   5. Contiguous Bundles：提高资产加载速度，新版本推荐直接勾上
   6. Max Concurrent Web Requests
   7. Group Hierarchy with Dashes：如果group是通过-分割，这里可以在编辑器上展开为目录
   8. Ignore Invalid、Unsupported Files in Build：这种一般是要扩展Provider
4. Build and Play Mode Scripts：这里是一些打包脚本，扩展的脚本要加在这里才能使用
5. AssetGroupTemplate：Group模板
6. InitializationObjects：初始化对象，用于和Addressable同时初始化的资源

#### Addressable Groups
管理资源组，同时提供了一些快捷菜单

Play Mode Script
1. Use Asset Database：直接使用AssetDatabase.Load加载，这个不会在event viewer里面呈现
2. Simulate Groups(advanced):模拟正式的ab，推荐使用
3. Use Existing Build(requires built groups)：使用正式ab加载

如果需要热更新
1. Tools/Check for content update
2. Update previous build

##### Schema
1. Content Packing & Loading
   1. Build and Load Paths:定义了打包路径&加载路径
   2. Include In Build：是否包含在构建中 
   3. Force Unique Provider:id唯一，并且只会处理Assets
   4. Use Asset Bundle Cache:勾选后悔尝试加载本地缓存的ab
   5. Use Asset Bundle crc
   6. Use Asset Bundle Crc For Cached Bundle
   7. Timeout  :UnityWebRequest超时时间
   8. Chunked Transfer :告诉UnityWebRequest是否以HTTP/1.1 chunked-transfer encoding method.？？？
   9. Redirect Limit :重定向次数
   10. RetryCount :重试次数
   11. BundleMode
       1.  Pack Together:这个组打一起
       2.  Pack Separately:单个文件一个bundle
       3.  Pack Together By Label:按标签打包
   12. BundleNameing
       1.  FileName
       2.  Append Hash to Filename
       3.  Use Hash of AssetBundle
       4.  Use Hash of Filename
   13. Asset Provider
       1.  <none>
       2.  Content Catalog Provider : Provider for content catalogs.  This provider makes use of a hash file to determine if a newer version of the catalog needs to be downloaded.
       3.  AssetBundle Provider : IResourceProvider for asset bundles.  Loads bundles via UnityWebRequestAssetBundle API if the internalId starts with "http".  If not, it will load the bundle via AssetBundle.LoadFromFileAsync.
       4.  Assets from AssetDatabase Provider : Provides assets loaded via the AssetDatabase API.  This provider is only available in the editor and is used for fast iteration or to simulate asset bundles when in play mode.
       5.  Sprites from Atlases Provider : Provides sprites from atlases
       6.  Assets from Bundles Provider : Provides assets stored in an asset bundle.
       7.  JSON Asset Provider : Converts JSON serialized text into the requested object.
       8.  Assets from Legacy Resources : Provides assets loaded via Resources.LoadAsync API.
       9.  Text Data Provider : Provides raw text from a local or remote URL.
       10. Virtual AssetBundle Provider : Simulates the loading behavior of an asset bundle.  Internally it uses the AssetDatabase API.  This provider will only work in the editor.
       11. Assets from Virtual Bundles
   14. Asset Bundle Provider 同上
       
2. Content Update Restriction
   1. static
   2. non-static
3. Resources and Built In Scenes
    一般不需要，包含内部的资源，交给默认的Build In Data组即可


#### Addressable Profiles
这里可针对不同的开发、正式环境做不同的配置

以下是默认存在的五个变量
1. BuildTarget  平台
2. LocalBuildPath 资源包构建的本地路径 (开发环境推荐)
3. LocalLoadPath 资源包加载的本地路径 
4. RemoteBuildPath 资源包构建的远程路径 (正式环境推荐打包到服务器，再拷贝到streammingAssets，通过重定向路径来做)
5. RemoteLoadPath 资源包加载的远程路径

##### Syntax
所有的变量类型都是string，这里有两种额外的语法支持
1. 中括号[]，这里面的变量是构建时获取的
2. 大括号{}，这里的变量是运行时获取的，那么，这里可通过这个路径与前面的RemoteLoadPath结合

#### Addressable Event Viewer
提供了查看加载、资源引用的编辑器
TODO

#### Addressable Analyze
提供了资源规则的检查逻辑
TODO

#### Addressable Hosting
提供了本地模拟服务器的能力，这里的VariableName 可以直接在前面的Profiles里面使用
TODO：疑似网络问题QAQ

#### 注意事项:
1. local non static 这种类型的group一旦变动，并且使用了Update a Previous Build，客户端会无法加载资源，官方也建议不使用
2. static类型的group一旦有变动，那么就会额外生成一个Content Update Change的组，不建议使用
3. 如果客户端不热更新(switch、stream)，那么就可以使用local static
4. 手游一般会使用remote non static，但这种不满足一开始将资源带入包的想法，还需要在调研扩展。
5. Addressables本身不对更新资源做版本号管理的，这个需要自己做。

#### 实现逻辑:
1. 启动时，检查热更新，下载资源
2. 调用Addressables.InitializeAsync()主动更新，防止第一次调用的时候，初始化卡顿
3. 打包的时候，生成的ab包在服务器目录，在程序发布的时候将资源包拷贝进streamingAssets中，加载时先检查本地时候有资源，且是否是最新的，如果没有就下载最新的ab包，注意catalog文件永远保持最新。
Addressable.InternalIdTransformFunc 重定向

``` 示例(https://blog.csdn.net/qq_14903317/article/details/108582372)
private string InternalIdTransformFunc(UnityEngine.ResourceManagement.ResourceLocations.IResourceLocation location)
{
    //判定是否是一个AB包的请求
    if (location.Data is AssetBundleRequestOptions)
    {
        //PrimaryKey是AB包的名字
        //path就是StreamingAssets/Bundles/AB包名.bundle,其中Bundles是自定义文件夹名字,发布应用程序时,复制的目录
        var path = Path.Combine(Addressables.RuntimePath, "Bundles", location.PrimaryKey);
        if (File.Exists(path))
        {
            return path;
        }
    }
    return location.InternalId;
}
```

TODO:
1. 本地环境profile与工程结合
2. 如何将首包资源打入apk，重定向加载
3. 热更新代码
4. Asset Provider
5. importer：https://github.com/favoyang/unity-addressable-importer
6. 引用计数
7. API https://docs.unity3d.com/Packages/com.unity.addressables@1.17/manual/BuildPlayerContent.html
   1. BuildPlayerContent
   2. DownloadDependenciesAsync
   3. ExceptionHandler
   4. InitializeAsync
   5. InstantiateAsync
   6. LoadContentCatalogAsync
   7. LoadingAddressableAssets
   8. LoadResourceLocationsASync
   9. LoadSceneAsync
   10. TransformInternalId
   11. UpdateCatalogs