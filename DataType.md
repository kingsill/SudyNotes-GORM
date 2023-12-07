[toc]
[gorm官方文档](https://gorm.io/zh_CN/docs/data_types.html)：自定义数据类型
# 自定义数据类型
&emsp;&emsp;数据空中很多情况下数据是多变的，我们这篇文章将以json和数组为例学习GORM的自定义数据类型方法。
&emsp;&emsp;自定义的数据类型必须实现 ``Scanner`` 和 ``Valuer`` 接口，以便让 GORM 知道如何将该类型接收、保存到数据库
&emsp;&emsp;通过两个接口将json、数组转换为字符串类型，其实际为序列化和反序列化的过程

## 自定义json
### 结构体定义
```go
// Info json的序列化与反序列化的实例，定义Info的信息，方便后续进行转化及查询
type Info struct {
	Status     string `json:"status"`
	Addr       string `json:"addr"`
	Age        int    `json:"age"`
	LiveOrDead bool   `json:"liveOrDead"`
}

// User 定义User表，表中的Info字段想要传入的即为json类型的数据
type User struct {
	Name string
	Info Info `gorm:"type:string"` //这里由于我们已经实现了Scanner和Valuer接口，当不属于基本数据类型的数据传入时，会自动调用这两个接口，自动赋予类型。当然我们这里也可以提前指定好，我们这里选择string类型
}
```
### Scaner和Valuer接口的实现
```go
// Scan 从数据库读取，将数据库中读取出来的数据类型还原为json,实现了sql.Scanner 接口
func (i *Info) Scan(value interface{}) error {

	v, _ := value.([]byte) //类型断言，断定为[]byte类型，我们在value方法中也是转换为[]byte类型输入到数据库中的

	var receiver Info
	err := json.Unmarshal(v, &receiver) //反序列化，将[]byte类型转化为我们需要的结构体
	if err != nil {
		return err
	}
	//fmt.Println(receiver)
	*i = receiver //将其内容传输给info

	return nil

}

// Value 存入数据库，将json转换为数据库可接受类型数据，实现dirver.Valuer接口
func (i Info) Value() (driver.Value, error) {

	return json.Marshal(i) //由结构体转换为json类型数据，返回[]byte

}
```
### 插入数据&查询数据 <span id="jump"></span>
生成（迁移）表格的方式不变，插入、查询记录的方式也与之前都相同
插入数据
```go
wang2 := User{Name: "wang2",
	Info: Info{
		Status:     "ok",
		Addr:       "zibo",
		Age:        18,
		LiveOrDead: true,
	}}

DB.AutoMigrate(&User{})
DB.Create(&wang2)
```
查询数据
```go
var QueryUser User
DB.Take(&QueryUser)//这里直接选择第一条内容作为演示
fmt.Printf("类型：%T\n内容：%v", QueryUser.Info, QueryUser)
//类型：main.Info
//内容：{wang2 {ok zibo 18 true}}
```
## 自定义数组
### 存储数组
#### json形式存储
&emsp;&emsp;比较方便的一种存储方式就是将数组转化为json类型的数据进行存储
&emsp;&emsp;大部分的代码都与json数据的存储一致，这里将所有代码贴在下面，并标注出不同
```go
package main

import (
	"database/sql/driver"
	"encoding/json"
	"fmt"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

var DB *gorm.DB

func init() {
	username := "root"
	password := "123456"
	host := "127.0.0.1"
	port := 3306
	Dbname := "gorm"
	timeout := "10s"

	dsn := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8mb4&parseTime=True&loc=Local&timeout=%s", username, password, host, port, Dbname, timeout)
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info),
	})
	if err != nil {
		fmt.Println("连接数据库失败, error=", err)
		return
	}
	DB = db
	fmt.Println("数据库连接成功")
}

type Ports [3]string//定义一个字符串类型的数组

type HostModel struct {
	ID    int
	IP    string
	Ports Ports
}
//前文json格式的存储，有不懂的可以参照这里所写的
/*// Info json的序列化与反序列化的实例，定义Info的信息，方便后续进行转化及查询
type Info struct {
	Status     string `json:"status"`
	Addr       string `json:"addr"`
	Age        int    `json:"age"`
	LiveOrDead bool   `json:"liveOrDead"`
}

// User 定义User表，表中的Info字段想要传入的即为json类型的数据
type User struct {
	Name string
	Info Info `gorm:"type:string"` //这里由于我们已经实现了Scanner和Valuer接口，当不属于基本数据类型的数据传入时，会自动调用这两个接口，自动赋予类型。当然我们这里也可以提前指定好，我们这里选择string类型
}*/

func main() {
	G15 := HostModel{
		ID:    1,
		IP:    "192.168.1.1",
		Ports: [3]string{"1", "2", "3"},
	}

	DB.AutoMigrate(&HostModel{})
	DB.Create(&G15)

	var QueryUser HostModel
	DB.Take(&QueryUser)
	fmt.Printf("类型：%T\n内容：%v", QueryUser.Ports, QueryUser)
	//类型：main.Info
	//内容：{wang2 {ok zibo 18 true}}
}

//注意Scan方法传入为指针，而value直接传入结构体
@@@@@@@    传入scan和value的形参是这唯一不同的    @@@@@@@@@@
// Scan 从数据库读取，将数据库中读取出来的数据类型还原为json,实现了sql.Scanner 接口
func (i *Ports) Scan(value interface{}) error {

	v, _ := value.([]byte) //类型断言，断定为[]byte类型，我们在value方法中也是转换为[]byte类型输入到数据库中的

	var receiver Ports
	err := json.Unmarshal(v, &receiver) //反序列化，将[]byte类型转化为我们需要的结构体
	if err != nil {
		return err
	}
	//fmt.Println(receiver)
	*i = receiver //将其内容传输给info

	return nil

}

// Value 存入数据库，将json转换为数据库可接受类型数据，实现dirver.Valuer接口
func (i Ports) Value() (driver.Value, error) {

	return json.Marshal(i) //由结构体转换为json类型数据，返回[]byte

}

```
#### 字符串存储
&emsp;&emsp;使用分割符进行数组数据的分割，这里只有scan和value函数的实现方法有区别，结构体的定义并不发生改变

>知识回顾：
&emsp;&emsp;切片的底层就是一个数组，所以我们可以基于``数组``通过``切片表达式``得到``切片``。 切片表达式中的low和high表示一个索引范围（左包含，右不包含），也就是下面代码中从数组a中选出1<=索引值<4的元素组成切片s，得到的切片长度=high-low，容量等于得到的切片的底层数组的容量。
&emsp;&emsp;为了方便起见，可以省略切片表达式中的任何索引。省略了low则默认为0；省略了high则默认为切片操作数的长度:
&emsp;&emsp;a[2:]  // 等同于 a[2:len(a)]
&emsp;&emsp;a[:3]  // 等同于 a[0:3]
&emsp;&emsp;a[:]   // 等同于 a[0:len(a)]
```go
// Scan 从数据库读取，将数据库中读取出来的数据类型还原为json,实现了sql.Scanner 接口
func (i *Ports) Scan(value any) error {

	//v := value.(string) //!!!!错误版本，？？？？类型断言，断定为[]byte类型，我们在value方法中也是转换为[]byte类型输入到数据库中的
	//panic: interface conversion: interface {} is []uint8, not string

	v := value.([]byte) //我们在Scan函数中value的断言都选择[]byte方法

	cache := strings.Split(string(v), "|") //将v转变为字符串之后用|分隔符分割,还原为[]string类型数据

	*i = cache
	//fmt.Println(cache)
	return nil

}

// Value 存入数据库，将json转换为数据库可接受类型数据，实现dirver.Valuer接口
func (i Ports) Value() (driver.Value, error) {
	return strings.Join(i, "|"), nil //由结构体转换为json类型数据，返回string
}
```

## 创建&查询数据
这里与[json数据的写法](#jump)一致
```go
G15 := HostModel{

	IP:    "192.168.1.1",
	Ports: []string{"1", "2", "3"},
}

DB.AutoMigrate(&HostModel{})
DB.Create(&G15)

var QueryUser HostModel
DB.Take(&QueryUser)
fmt.Printf("类型：%T\n内容：%v", QueryUser.Ports, QueryUser)
```