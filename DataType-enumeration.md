[toc]
学习来源：[枫枫知道](https://docs.fengfengzhidao.com/#/docs/gorm%E6%96%87%E6%A1%A3/10.%E8%87%AA%E5%AE%9A%E4%B9%89%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B?id=%e6%9e%9a%e4%b8%be%e7%b1%bb%e5%9e%8b)
# 枚举
&emsp;&emsp;很多时候，由于字符串的所占据空间较大，而某些状态的值是一定的，所以我们希望用枚举来固定对应的值。这样不仅可以节省空间，更关键的是可以方便后期的维护
&emsp;&emsp;我们这里以主机管理为例，状态有 Running 运行中， OffLine 离线， Except 异常
&emsp;&emsp;我们跳过原文章中方法逐步优化的过程，直接到最完善的版本，即使用数字表示状态和使用类型别名的方法，最后通过手动逻辑方法来实现golang中的枚举
## 枚举的终极办法，原文3.0版本
```go
package main

import (
	"encoding/json"
	"fmt"
)

type Status int // 类型别名

// 定义running、except、offline对应的常量，本质为整数数据
// 通过定义常量我们可以在查询时直接返回这里的常量名，而不是数字等容产生歧义
const ( //在线、异常、离线
	Running Status = iota + 1 // iota为golang中常量的行索引，起步为0，每行+1
	Except
	Offline
)

type Host struct {
	ID     uint   `gorm:"json:id"`
	Name   string `gorm:"json:name"`
	Status Status `gorm:"json:status"`
}

// 在status的json序列化过程中进行常量和字符串的转化
func (status Status) MarshalJSON() ([]byte, error) {
	var str string //定义一个字符串方便对status的json转换

	switch status {
	case Running:
		str = "running"
	case Except:
		str = "except"
	case Offline:
		str = "offline"
	}

	return json.Marshal(str)

}

func main() {
	//定义一个host主机实例
	var host = Host{
		Status: Running,
		Name:   "wang2",
	}

	result, _ := json.Marshal(host) //序列化host为json以存储

	fmt.Printf(string(result))
	//查询结果如下，可以看到status的值这里查询为字符
	//{"ID":0,"Name":"wang2","Status":"running"}
}

```
### 枚举的实现举例
在需要输出的时候（print，json），自定义类型就变成了字符串
>复习阶段：数据类型的转换
Go 语言类型转换基本格式如下：
>```go
>type_name(expression)
>```
>type_name 为类型，expression 为表达式。
字符串类型的数据转换可以用到strconv,strconv.atoi将字符串转换为int;strconv.itoa将int转换为字符串
```go
package main

import (
	"encoding/json"
	"fmt"
	"time"
)

type WeekDay int //定义一个数值的类型别名，方便显示

// 定义weekday与整数的对应
const (
	Sunday WeekDay = iota + 1
	Monday
	Tuesday
	Wednesday
	Thursday
	Friday
	Saturday
)

var WeekStringList = []string{"Sunday", "Monday", "Tuesday", "Thursday", "Wednesday", "Friday", "Saturday"}
var WeekTypeList = []WeekDay{Sunday, Monday, Tuesday, Thursday, Wednesday, Friday, Saturday}

// 通过定义weekday的string方法将其本质int映射为对应的string 自定义》字符
// 实现了自定义weekday类型的string接口，会在输出、打印weekday时自动调用，其被包含在stringer包中，具体可查阅该包
func (W WeekDay) String() string {
	s := WeekStringList[W]
	//fmt.Println("from string 方法", s)
	return s
}

func (W WeekDay) MarshalJSON() ([]byte, error) {
	s, _ := json.Marshal(W.String())
	//fmt.Println("marshal启用")
	return s, nil

}

// ParseStrWeekday 将字符串类型转化为自定义类型 字符串》自定义
func (W *WeekDay) ParseStrWeekday(week string) {
	for i, v := range WeekStringList {
		if v == week {
			*W = WeekTypeList[i]
			fmt.Println(*W)
		}
	}
}

// ParseIntWeekday Int》weekday自定义类型
func (W *WeekDay) ParseIntWeekday(week int) {
	*W = WeekTypeList[week-1]
}

type DayInfo struct {
	WeekDay WeekDay   `json:"WeekDay"`
	Data    time.Time `json:"Data"`
}

func main() {
	December7th := DayInfo{
		WeekDay: Wednesday,
		Data:    time.Now(),
	}
	re, _ := json.Marshal(December7th)

	fmt.Println(string(re))

	//通过ParseStrWeekday和ParseIntWeekday这两个方法，实现了输入字符串和int都可以转换为weekday类型，同时通过该类型的string接口的实现可以在输出时自动以对应的字符串了类型进行输出
	var w WeekDay
	w.ParseStrWeekday("Sunday")
	fmt.Println("我输入的字符串“Sunday”，输出结果为：", w)

	w.ParseIntWeekday(2)
	fmt.Println("我输入的int类型 2，输出结果为： ", w)

}

```