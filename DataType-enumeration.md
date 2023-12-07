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
```go
package main

import (
	"encoding/json"
	"fmt"
	"time"
)

type WeekDay int //定义一个数值的类型别名，方便显示

var WeekStringList = []string{"Sunday", "Monday", "Tuesday", "Thursday", "Wednesday", "Friday", "Saturday"}
var WeekTypeList = []WeekDay{Sunday, Monday, Tuesday, Thursday, Wednesday, Friday, Saturday}

func (WeekDay WeekDay) String() string {

	s := WeekStringList[WeekDay-1]
	fmt.Println(s)
	return s
}

func (WeekDay WeekDay) MarshalJSON() ([]byte, error) {

	s, _ := json.Marshal(WeekDay.String())
	return s, nil
}

// 超级落后
const (
	Sunday WeekDay = iota + 1
	Monday
	Tuesday
	Wednesday
	Thursday
	Friday
	Saturday
)

/*
	func (WeekDay WeekDay) MarshalJSON() ([]byte, error) {
		var str string
		switch WeekDay {
		case Sunday:
			str = "Sunday"
		case Monday:
			str = "Monday"
		case Tuesday:
			str = "Tuesday"
		case Thursday:
			str = "Thursday"
		case Wednesday:
			str = "Wednesday"
		case Friday:
			str = "Friday"
		case Saturday:
			str = "Saturday"
		}
		return json.Marshal(str)
	}
*/

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

	fmt.Printf(string(re))
}

```