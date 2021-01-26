# Thrift之IDL

Thrift是Facebook开发的跨语言平台的一种RPC服务框架（跨语言的还有gRPC），利用IDL（Interface Description Language）文件来定义接口和数据类型，通过Thrift提供的编译器编译成不同语言代码，以此实现跨语言调用。

## 基本类型（Base Types）

|Type|Desc|JAVA|GO|
|:-:|:-:|:-:|:-:|
|i8|有符号的8位整数|byte|int8|
|i16|有符号的16位整数|float|int16|
|i32|有符号的32位整数|int|int32|
|i64|有符号的64位整数|long|int64|
|double|64位浮点数|double|float64|
|bool|布尔值|boolean|bool|
|string|文本字符串（UTF-8编码格式）|java.lang.String|string|

## 特殊类型（Special Types）

**binary**: 未编码的字节序列，是一string的一种特殊形式；这种类型主要是方便某些场景下JAVA调用。JAVA中对应的是java.nio.ByteBuffer类型，GO中是[]byte。

## 集合容器（Containers）

|Type|Desc|JAVA|GO|remark|
|:-:|:-:|:-:|:-:|:-:|
|list\<T>|元素有序列表，允许重复|java.util.List|[]T||
|set\<T>|元素无序列表，不允许重复|java.util.Set|[]T|GO没有set集合以数组代替|
|map<K,V>|key-value结构数据，key不允许重复|java.util.Map|map[K] V||
>Tips: 在使用容器类型时必须指定泛型，否则无法编译idl文件。其次，泛型中的基本类型，JAVA语言中会被替换为对应的包装类型。

## 常量及类型别名（Const&&Typedef）

```thrift
//常量定义
const i32 MALE_INT = 1
const map<i32, string> GENDER_MAP = {1: "male", 2: "female"}
//某些数据类型比较长可以用别名简化
typedef map<i32, string> gmp
```

## 枚举（enum）

某些方法需要限制参数范围的时候可以使用enum，枚举是一种构造数据类型。

* 变量只能赋值整数，可以为负数；
* 变量也可以使用16进制整数赋值；
* 变量默认从0开始赋值；
* 变量分隔符可使用（,）和（;），可以混用也可均不用;建议统一；

```thrift
//变量默认赋值从0开始；
enum GenderEnum{
    UNKNOWN,//0
    MALE,//1
    FEMALE//2
}
//第一个变量赋值为1，后面的变量一次递增；
enum RoleEnum{
    WARIOR = 1,//战吊1
    MAGE,//法爷2
    WARLOCK,//术士3
    PRIEST,//牧师4
    DRUID//德鲁伊5
}
```

## 结构体（Structs） :id=struct

日常开发中，基本类型和集合类型无法满足业务需要；这时可以使用struct关键字来自定义类型，本质等同于OOP语言中的类（如JavaBean）。

**struct使用需要注意以下几点:**

* struct没有继承关系，可以嵌套引用包括自己（有些文章说不能嵌套）；
* 成员字段是强类型（即明确指定类型），字段不允许重复；
* 字段必须使用正整数+冒号编号（如 1: i32 id），编号不允许重复，可以不连续；
* 成员字段间分隔符可使用（,）和（;），可以混用也可均不用，为了方便阅读和该死的强迫症，建议统一；
* 字段可以使用optional和required修饰，默认是optional；区别是required修饰的字段在远程调用时必须传值，optional修饰的字段只有在被赋值时才会被序列化；
* 基本类型和enum类型的字段可以默认赋值，但struct类型的字段不允许；
* 基于业务域划分或设计角度，多个struct可以定义在不同的idl文件中，通过include “xxx.thrift”引用;
  注意使用include引入的struct时需要使用文件名前缀(n: xxx.DemoStruct demoStruct), 编译时thrift自动补全被引入thrift文件中定义的namespace。

```thrift
/**
 * struct嵌套及include使用示例，备注错误的位置会导致IDL文件无法编译，具体请参考下方的”struct不可以双向引用“。
 */
//guild.thrift
namespace java io.buman.guild
include "player.thrift" //错误
struct Guild {
    10: i32 id,//公会ID
    20: string name，//公会名称
    30: player.Player president //错误
}
//player.thrift
namespace java io.buman.player
include "guild.thrift"

//变量默认赋值从0开始；
enum GenderEnum{
    UNKNOWN,//0
    MALE,//1
    FEMALE//2
}

//第一个变量赋值为1，后面的变量一次递增；
enum RoleEnum{
    WARIOR = 1,//战吊1
    MAGE,//法爷2
    WARLOCK,//术士3
    PRIEST,//牧师4
    DRUID//德鲁伊5
}

struct Player {
    1: i32 id,
    2: required string name,
    3: required RoleEnum role,
    4: required GenderEnum gender = GenderEnum.UNKNOWN,
    5: Player cp,
    6: guild.Guild guild
}
```

>Tips: 字段用required修饰时，编译的后的JAVA代码通过validate()校验。

```java
  public void validate() throws org.apache.thrift.TException {
    // check for required fields
    if (name == null) {
      throw new org.apache.thrift.protocol.TProtocolException("Required field 'name' was not present! Struct: " + toString());
    }
    if (role == null) {
      throw new org.apache.thrift.protocol.TProtocolException("Required field 'role' was not present! Struct: " + toString());
    }
    if (gender == null) {
      throw new org.apache.thrift.protocol.TProtocolException("Required field 'gender' was not present! Struct: " + toString());
    }
    // check for sub-struct validity
    if (guild != null) {
      guild.validate();
    }
  }
```

>Tips: struct不可以双向引用

```bash
//player.thrift文件引用了guild.thrfit后，guild.thrift文件再引用playe.thrift文件编译时循环扫描文件无法通过。 
(base) buman@bogon % thrift --gen java player.thrift
zsh: segmentation fault  thrift --gen java player.thrift
(base) buman@bogon % thrift --gen -v java player.thrift //用--verbose或者-v再试一下 
Scanning player.thrift for includes
Scanning guild.thrift for includes
Scanning player.thrift for includes
Scanning guild.thrift for includes
Scanning player.thrift for includes
... ...
zsh: segmentation fault  thrift --gen java -v player.thrift
```

## 异常（Exceptions)

thrift支持在[服务](#service)里面使用exception关键字自定义异常，结构上等同于[结构体](#struct)；便于客户端更好的识别和处理服务的各种异常状况。

```thrift
exception PlayerNotFoundException {
    1: i32 code = 400,
    2: string msg
}
```

## 服务（Services） :id=service

用service关键字来定义服务接口，它是一组命名函数的集合；idl的语法都是为了服务service的定义。

* 命名函数具有参数列表和返回值类型，参数列表可以为空，没有返回值使用void关键字修饰；
* 参数列表类似struct中的成员字段定义，每个参数前都需要正整数+冒号编号，编号可以不连续；
* 参数列表可以使用分隔符（,）和（;），可以混用也可均不用（自由就是造孽）；习惯使用（,）;

```thrift
service PlayerService {

    /**
     * 玩家注册
     */
    Player signIn(1: Player player)

    /**
     * 查询所有玩家
     */
    list<Player> queryAllPlayer()

    /**
     * 注册公会
     */
    guild.Guild registerGuild(1: guild.Guild guild)

    /**
     * 查询所有公会
     */
    list<guild.Guild> queryAllGuild()

    /**
     * 为玩家添加cp
     */
    Player addCp(1: i32 pid, 2: Player player) throws (1: PlayerNotFoundException e)

    /**
     * 玩家加入公会
     */
    Player joinGuild(1: i32 pid, 2: i32 gid) throws (1: PlayerNotFoundException e)
}
```

>Tips: 注意exception使用的语法，也是需要添加编号的。

## thrift编译环境安装

[官网下载安装https://thrift.apache.org/download.html](https://thrift.apache.org/download.html)

```bash
//macOS
brew install thrift
```

## idl文件编译

```bash
//编译JAVA语言文件，默认输出到当前目录，生成gen-java文件夹（gen-{编译语言}）；
thrift --gen java playler.thrift
thrift --gen java guild.thrift
//指定输出目录
thrift --gen java -o target player.thrift
thrift --gen java -o target guild.thrift
```

>Tips: 注意idl文件中include的struct不会被编译，需要单独编译每个idl文件。

## 示例代码

[https://github.com/bumangrowing/thrift-demo-java/tree/1.0](https://github.com/bumangrowing/thrift-demo-java/tree/1.0)

## 参考

* [1] [官方文档（https://thrift.apache.org/docs/types）](https://thrift.apache.org/docs/types)
