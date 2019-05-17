#### 1.序列化
序列化是将对象或对象图转换成字节流的过程。反序列化正好相反。

SerializableAttribute这个定制特性智能应用与引用类型、值类型、枚举类型和委托类型。枚举和委托类型总是可序列化的，所以不用显式应用该特性。SerializableAttribute不会被派生类型继承。
NorSerializedAttribute定制特性指出类型中不应序列化的字段。
```
OnSerializedAttribute
OnDeserializedAttribute
OnSerializingAttribute
OnDeserializingAttribute
```
序列化一组对象时，格式化器首先调用对象标记了OnSerializing特性的所有方法，然后序列化对象的所有字段。最后调用对象标记了OnSerialized特性的所有方法

OptionalFieldAttribute特性，类型中新加的每个字段都要引用该特性，格式化器看到这个特性应用于一个字段时，不会因为流中的数据不包含这个字段而抛出SerializationException

#### 2.格式化器序列化类型实例
为了简化格式化器的操作，FCL在System.Runtime.Serialization命名空间提供了一个FormatterServices类型。该类型只包含静态方法，而且不能被实例化。
序列化
1、格式化器调用FormatterServices的GetSerializableMembers方法：
```
public static MemberInfo[] GetSerializableMembers(Type type, StreammingContext context);
```
该方法利用反射获取类型的public和private实例字段。
2、对象被序列化，MemberInfo对象数组传给FormatterServices的静态方法GetObjectData
```
public static Object[] GetObjectData(Object obj, MemberInfo[] members);
```
3、格式化器将程序集标识和类型的完整名称写入流中
4、格式化器遍历两个数组中的元素，将每个成员的名称和值写入流中

反序列化
1、格式化器中流中读取程序集标识和完整类型名称。如果程序集未加载到AppDomain中就先加载它。如果不能加载抛SerializationException异常，对象不能反序列化。如果已加载，格式化器将程序集标识信息和类型全名传给FormatterServices的静态方法GetTypeFromAssembly
public static Type GetTypeFromAssembly(Assembly assem, String name);
返回Type对象代表反序列化的哪个对象的类型
2、格式化器调用FormatterServices的GetUninitalizedObject方法
```
public static Object GetUnitializedObject(Type type);
```
为一个新对象分配内存，但不调用构造器。所有直接都被初始化为null或0
3、格式化器构造版并初始化一个MemberInfo数组。也是调用前面的GetSerializableMembers方法。这个方法返回序列化好、现在需要反序列化的一组字段
4、格式化器根据流中包含的数据创建并初始化一个Object数组
5、将分配对象、MemberInfo数组以及冰心Object数组（其中包含字段值）的引用传给FormatterServices的PopulateObjectMembers方法
```
public static Object PopulateObjectMembers(Object obj, MemberInfo[] members, Object[] data);
```
这个方法遍历数组，将每个字段初始化为对应的值，至此，对象彻底被反序列化了。

#### 3.控制序列化/反序列化的数据
前面的特性不能提供想要的全部控制。
格式化器内部使用的是反射，反射较慢，为了对序列化/反序列化进行完全的控制，并避免使用反射，可使接口实现System.Runtime.Serialization.ISerializable接口
```
public interface ISerializable {
    void GetObjectData(SerializationInfo info, StreamingContext context);
}
```
其他接口可能调用此方法，并传入损坏的数据，可以给此方法添加一下特性
```
[SecurityPermissionAttribute(SecurityAction.Demand, SerializationFormatter = true)]
```
格式化器序列化对象图会检查每个对象，如果实现了ISerializable接口，就会忽略所有定制特性，改为构造新的System.Runtim.Serialization.SerializationInfo对象

构造SerializationInfo对象时，格式化器要传递两个参数:Type和System.Runtime.Serialization.IFormatterConverter
1. Type参数标识要序列化的对象。唯一性的标识一个类型需要两个部分的信息
   
   * 类型的字符串名称及其程序集标识(包括程序集名、版本、语言文化和公钥)
   * 构造好的SerializationInfo对象包含类型的全名(内部查询Type的FullName),这个字符串会存储到一个私有字段中(如果需要获取类型的全名，可查询SerializationInfo的FullTypeName属性)。类似的，获取类型的定义程序集(通过内部查询Type的Module字段，在查询Module的Assembly属性，再查询Assembly的FullName属性,SerializationInfo的AssemblyName属性)
2. 格式化器调用类型的GetObjectData方法，传递它对SerializationInfo的引用
    GetObjectData方法决定需要哪些信息来序列化对象，并将这些信息添加到SerializationInfo对象中。
    GetObjectData调用SerializaationInfo类型提供的AddValue方法来指定序列化的信息，针对每个数据都要调用一次AddValue

以下代码展示了Dictionary<TKey, TValue>类型如何实现ISerializable和IDeserializationCallback接口来控制其对象的序列化和反序列化
```
[Serializable]
public class Dictionary<TKey, TValue> : ISerializable, IDeserializationCallback
{
    private SerializationInfo m_siInfo; // 只用于反序列化

    // 用于控制反序列化的特殊构造器，这是ISerializable需要的
    [SecurityPermission(SecurityAction.Demand, SerializationFormatter = true)]
    protected Dictionary(SerializationInfo info, StreamingContext context)
    {
        m_siInfo = info;
    }
    // 用于控制序列化的方法
    [SecurityCritical]
    public void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        info.AddValue("Version", m_version);
        info.AddValue("Compare", m_compare, typeof(IEqualityComparer<TKey>));
        info.AddValue("HashSize", (m_buckets == null) ? 0 : m_buckets.Length));
        if (m_buckets != null)
        {
            KeyValuePair<TKey, TValue> array = new KeyValuePair<TKey, TValue>(Count);
            CopyTo(array, 0);
            info.AddValue("KeyValuePair", array, typeof(KeyValuePair<TKey, TValue>));
        }
    }

    // 所有key/value对象按都反序列化后调用方法
    public void OnDeserialization(object sender)
    {
        if (m_siInfo == null) return; // 从不设置，直接返回

        Int32 num = m_siInfo.GetInt32("Version");
        Int32 num2 = m_siInfo.GetInt32("HashSize");
        m_compare = (IEqualityComparer<TKey>)m_siInfo.GetValue("Compare", typeof(IEqualityComparer<TKey>));
        m_buckets = new Int32[num];
        // ...
    }
}

```

注意代码的可访问性与安全性。无论这个特殊构造器是如何声明的，格式化器都能调用。

#### 4.要实现ISerializable但基类未实现

```
[Serializable]
internal class Derived : Base, ISerializable
{
    private DateTime m_date = DateTime.Now;
    public Derived() { /* Make the type instantiable */} // 可实例化

    // 如果这个构造器不存在，便会引发一个SerializationException异常
    [SecurityPermission(SecurityAction.Demand, SerializationFormatter = true)]
    private Derived(SerializationInfo info, StreamingContext context)
    {
        // 为我们的类和基类获取可序列化的成员集合
        Type baseType = this.GetType().BaseType;
        MemberInfo[] mi = FormatterServices.GetSerializableMembers(baseType, context);
        // 从info对象反序列化基类的字段
        for(int i = 0; i < mi.Length; i++)
        {
            // 获取字段，并设置为反序列化的值
            FieldInfo fi = (FieldInfo)mi[i];
            fi.SetValue(this, info.GetValue(baseType.FullName + "+" + fi.Name, fi.FieldType));
        }
        m_date = info.GetDateTime("Date");
    }

    [SecurityPermission(SecurityAction.Demand, SerializationFormatter = true)]
    public virtual void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        // 为这个类序列化希望的值
        info.AddValue("Data", m_date);
        // 获取我们的类和基类的可序列化的成员
        Type baseType = this.GetType().BaseType;
        MemberInfo[] mi = FormatterServices.GetSerializableMembers(baseType, context);
        // 将基类的字段序列化到info对象中
        for(int i = 0; i < mi.Length; i++)
        {
            // 为字段名附加基类全名作为前缀
            info.AddValue(baseType.FullName + "+" + mi[i].Name, ((FieldInfo)mi[i]).GetValue(this));
        }
    }
}
```

#### 5.流上下文
StreamingContext 一个非常简单的值类型
成员:
   
   1. State StreamingContextStates 一组位标志，指定要序列化/反序列化的对象的来源或目的地
   2. Context Object               一个对象引用，对象中包含用户希望的任何上下文信息

![24-01](../Pictures/CLR_via_C_Sharp/24_01.png)

IFormatter接口(同时由BinaryFormatter和SoapFormatter类型实现)定义了StreamContext类型的可读写属性Context。构造格式化器时，格式化器会初始化它的Context属性，将State设为All，并将对额外状态对象的引用置为null
格式化器构造好后，就可以使用任何StreammingContextStates为标志来构造一个StreamingContext结构，并可选择传递一个对象引用。在调用格式化器的Serialize或Deserialize方法之前只需要将格式化器的Context属性设为这个新的StreamingContext对象

#### 6.序列化代理
格式化器允许不是"类型实现的一部分"的代码重写该类型"序列化和反序列化对象"的方式。应用程序代码之所以要重写类型的行为，主要是出于以下方面的考虑
1. 允许开发人员序列化最初没有设计成要序列化的类型
2. 允许开发人员提供一种方式将类型的一个版本映射到类型的一个不同的版本

序列化代理类型必须实现System.Runtime.Serialization.ISerializationSurrogate接口
```
public interface ISerializationSurrogate{
    void GetObjectData(Object obj, SerializationInfo info, StreammingContext context);
    Object SetObjectData(Object obj, SerializationInfo info, StreammingContext, context, ISurrogateSelector selector);
}
```
例子，程序含有一些DateTime对象，其中包含用户计算机的本地值。但如果将DateTime对象序列化到流中，同时希望值以国际标准时间序列化。
```
internal sealed class UniversalToLocalTimeSerializationSurrogate : ISerializationSurrogate {
    public void GetObjectData(Object obj, SerializationInfo info, StreammingContext, context) {
        // 将DateTime从本地时间转换成UTC
        info.AddValue("Date", ((DateTime)obj)).ToUniversalTime().ToString("u");
    }

    public Object SetObjectData(Object obj, SerializationInfo info, StreammingContext context) {
        // 将DateTime从UTC转换成本地时间
        return DateTime.ParseExact(info.GetValue("Date"), "u", null).ToLocalTIme();
    }
}
```
以下代码对UniversalToLocalTimeSerializationSurrogate类进行测试
```
private static void SerializationSurrogateDemo() {
    using (var stream = new MemoryStream()) {
        // 1.构造所需的格式化器
        IFormatter formatter = new SoapFormatter();
        // 2.构造一个SurrogateSelector(代理选择器)对象
        SurrogateSelector ss = new SurrogateSelector();
        // 3.告诉带你选择器为DateTime使用我们的代码
        ss.AddSurrogateSelector(typeof(DateTime), formatter.Context, new UniversalToLocalTimeSerializationSurrogate());
        // 注意：AddSurrogate可多次调用来登记多个代理

        // 4.告诉格式化器使用代理选择器

        // 创建一个DateTime来代表当前机器的本地时间，并序列化
        DateTime localTime = DateTime.Now;
        formatter.Serialize(stream, localTime);
        stream.Position = 0;
        Console.WriteLine(new StreamReader(stream).ReadToEnd());
        stream.Position = 0;
        DateTime afterTime = (DateTime)formatter.Deserialize(stream);
        Console.WriteLine(afterTime);
    }
}
```
多个SurrogateSelector对象可链接到一起。例如：一个对象维护一组序列化处理，这些surrogate用于将类型序列化成proxy，以便通过网络传送或者跨AppDomain传送。还可以让另一个维护一组学历恶化处理，这些序列化代理用于将版本1的类型转换成版本2的类型

如果有多个希望格式化器使用的SurrogateSelecotr对象，必须将他们链接到一个链表中。SurrogateSelector类型实现了ISurrogateSelector接口，该接口定义了三个方法，都与链接有关
```
public interface ISurrogateSelector {
    void ChainSelector(ISurrogateSelector selector);
    ISurrogateSelector GetNextSelector();
    ISerializationSurrogate GetSurrogate(Type type, StreamingContext context, out ISurrogateSelector selector);
}

ChainSelector:在当前操作的ISurrogateSelector对象（this对象）之后插入
GetNextSelector：返回对链表下一个ISurrogateSelector对象的引用
GEtSUrrogate:在this对象中查找一对Type/StreammingContext。没找到就访问链表的下一个ISurrogateSelector对象，以此类推。返回的ISerializationSurrogate对找到的类型进行序列化/反序列化。
// ISurrogateSelector 包含匹配项的ISurrogateSelector对象
```
#### 7.反序列化对象时重写程序集/类型
序列化对象时，格式化器输出类型及其定义程序集的全名。反序列化时，格式化器根据这个信息确定要为对象构造并初始化什么类型
列举情形：
1. 开发把一个类型实现从一个程序集移动到另一个程序集(像程序集版本号变化)
2. 服务器对象序列化到发送到客户端的流中。客户端处理流时，可以将对象反序列化为完全不同的类型，该类型的代码知道如何向服务器的对象发出远程方法调用
3. 开发创建了类型的新版本，想将已序列化的对象反序列化类型的新版本
```
System.Runtime.Serialization.SerializationBinder类
```
以下假定版本1定义了Ver1的类，程序集的新版本定义了Ver1ToVer2SerializationBinder类和Ver2的类
```
class Ver1ToVer2SerializationBinder : SerializationBinder {
    public override Type BindToType(String assemblyName, String typeName) {
        // 将任何Ver1对象从版本1反序列化成一个Ver2对象

        // 计算定义Ver1类型的程序集名称
        AssemblyName assemVer1 = Assembly.GetExecutingAssembly().GetName();
        assemVer1.Version = new Version(1, 0, 0, 0)
        // 如果从v1.0.0.0反序列化Ver1对象，就把它转变成一个Ver2对象
        if (assemblyName == assemVer1.ToString() && typeName == "Ver1")
            return typeof(Ver2);
        // 后者就只返回请求的同一个类型
        return Type.GetType(String.Format("{0}, {1}", typeName, assemblyName));
    }
}
```
1. 构造好格式化器后，构造Ver1ToVer2SerializationBinder的实例，并设置格式化器的可读/可写属性Binder，让它引用绑定器(binder)对象。
2. 调用格式化器的Deserialize方法。反序列化期间，格式化器发现已设置了个绑定器，每个对象要序列化时，格式化器都调用绑定器的BindToType方法，向它传递程序集名称以及格式化器想要反序列化的类型

SerializationBinder还可重写BindToName方法，从而序列化对象时更改程序集/类型信息
```
public virtual void BindToName(TypeSerializedType, out String assemblyName, out string typeName);
```
序列化期间，格式化器调用这个方法，传递它想要序列化的类型。不传就是默认。
