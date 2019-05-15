#### 1.序列化
序列化是将对象或对象图转换成字节流的过程。反序列化正好相反。

SerializableAttribute这个定制特性智能应用与引用类型、值类型、枚举类型和委托类型。枚举和委托类型总是可序列化的，所以不用显式应用该特性。SerializableAttribute不会被派生类型继承。
NorSerializedAttribute定制特性指出类型中不应序列化的字段。

OnSerializedAttribute
OnDeserializedAttribute
OnSerializingAttribute
OnDeserializingAttribute

序列化一组对象时，格式化器首先调用对象标记了OnSerializing特性的所有方法，然后序列化对象的所有字段。最后调用对象标记了OnSerialized特性的所有方法

OptionalFieldAttribute特性，类型中新加的每个字段都要引用该特性，格式化器看到这个特性应用于一个字段时，不会因为流中的数据不包含这个字段而抛出SerializationException

#### 2.格式化器序列化类型实例
为了简化格式化器的操作，FCL在System.Runtime.Serialization命名空间提供了一个FormatterServices类型。该类型只包含静态方法，而且不能被实例化。
序列化
1、格式化器调用FormatterServices的GetSerializableMembers方法：
public static MemberInfo[] GetSerializableMembers(Type type, StreammingContext context);
该方法利用反射获取类型的public和private实例字段。
2、对象被序列化，MemberInfo对象数组传给FormatterServices的静态方法GetObjectData
public static Object[] GetObjectData(Object obj, MemberInfo[] members);
3、格式化器将程序集标识和类型的完整名称写入流中
4、格式化器遍历两个数组中的元素，将每个成员的名称和值写入流中

反序列化
1、格式化器中流中读取程序集标识和完整类型名称。如果程序集未加载到AppDomain中就先加载它。如果不能加载抛SerializationException异常，对象不能反序列化。如果已加载，格式化器将程序集标识信息和类型全名传给FormatterServices的静态方法GetTypeFromAssembly
public static Type GetTypeFromAssembly(Assembly assem, String name);
返回Type对象代表反序列化的哪个对象的类型
2、格式化器调用FormatterServices的GetUninitalizedObject方法
public static Object GetUnitializedObject(Type type);
为一个新对象分配内存，但不调用构造器。所有直接都被初始化为null或0
3、格式化器构造版并初始化一个MemberInfo数组。也是调用前面的GetSerializableMembers方法。这个方法返回序列化好、现在需要反序列化的一组字段
4、格式化器根据流中包含的数据创建并初始化一个Object数组
5、将分配对象、MemberInfo数组以及冰心Object数组（其中包含字段值）的引用传给FormatterServices的PopulateObjectMembers方法
public static Object PopulateObjectMembers(Object obj, MemberInfo[] members, Object[] data);
这个方法遍历数组，将每个字段初始化为对应的值，至此，对象彻底被反序列化了。

#### 3.控制序列化/反序列化的数据
前面的特性不能提供想要的全部控制。
格式化器内部使用的是反射，反射较慢，为了对序列化/反序列化进行完全的控制，并避免使用反射，可使接口实现System.Runtime.Serialization.ISerializable接口
public interface ISerializable {
void GetObjectData(SerializationInfo info, StreamingContext context);
}
其他接口可能调用此方法，并传入损坏的数据，可以给此方法添加一下特性
[SecurityPermissionAttribute(SecurityAction.Demand, SerializationFormatter = true)]

格式化器序列化对象图会检查每个对象，如果实现了ISerializable接口，就会忽略所有定制特性，改为构造新的System.Runtim.Serialization.SerializationInfo对象
