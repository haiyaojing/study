#### AddressableAssetSettings
Addressable的设置文件
1. profile in use: 这个可以选择配置(一般需要有编辑器下、发布两种环境)
2. Catalog
   1. Disable Catalog Update on Startup:这个勾选了等于说要自己去写下载逻辑，手游一般要勾选
   2. Player Version Override:TODO
   3. Compress Local Catalog:压缩，一般不勾选
   4. Optimize Catalog Size:TODO
   5. Build Remote Catalog: 是否打服务器包，勾选了会出现以下BuildPath和LoadPath
      1. BuildPath
      2. LoadPath
3. General
   1. Send Profiler Events:勾选后在Event Viewer里面可以看到资源加载情况、引用情况
   2. Log Runtime Exceptions
   3. Custom certificate handler：TODO
   4. Unique Bundles IDs：TODO
   5. Contiguous Bundles：TODO
   6. Max Concurrent Web Requests
   7. Group Hierarchy with Dashes：TODO
   8. Ignore Invalid、Unsupported Files in Build：要研究下其他类型的资源怎么搞TODO
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

实现逻辑:
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
4. importer：https://github.com/favoyang/unity-addressable-importer
5. 引用计数
6. API https://docs.unity3d.com/Packages/com.unity.addressables@1.17/manual/BuildPlayerContent.html
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