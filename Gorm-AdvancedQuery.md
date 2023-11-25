# 初始化表格内数据
```go

package main

import (
	"fmt"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

var DB *gorm.DB

type Student struct {
	ID     int     `gorm:"size:10"`
	Name   string  `gorm:"size:10"`
	Age    int     `gorm:"size:3"`
	Email  *string `gorm:"size:128"`
	Gender bool
}

func init() {
	username := "root"   //账号
	password := "123456" //密码
	host := "127.0.0.1"  //数据库地址，可以是Ip或者域名
	port := 3306         //数据库端口
	Dbname := "gorm"     //数据库名
	timeout := "10s"     //连接超时，10秒

	// root:root@tcp(127.0.0.1:3306)/gorm?
	dsn := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8mb4&parseTime=True&loc=Local&timeout=%s", username, password, host, port, Dbname, timeout)
	//连接MYSQL, 获得DB类型实例，用于后面的数据库读写操作。
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info),
	})
	if err != nil {
		panic("连接数据库失败, error=" + err.Error())
	}
	DB = db
	// 连接成功
	fmt.Println(db)
}

func main() {
	var studentList []Student

	DB.AutoMigrate(&Student{}) //创建或重新生成表格

	DB.Find(&studentList).Delete(&studentList) //删除表格中原有的内容

	studentList = []Student{ //初始数据定义，实例化
		{ID: 1, Name: "李元芳", Age: 32, Email: PtrString("lyf@yf.com"), Gender: true},
		{ID: 2, Name: "张武", Age: 18, Email: PtrString("zhangwu@lly.cn"), Gender: true},
		{ID: 3, Name: "枫枫", Age: 23, Email: PtrString("ff@yahoo.com"), Gender: true},
		{ID: 4, Name: "刘大", Age: 54, Email: PtrString("liuda@qq.com"), Gender: true},
		{ID: 5, Name: "李武", Age: 23, Email: PtrString("liwu@lly.cn"), Gender: true},
		{ID: 6, Name: "李琦", Age: 14, Email: PtrString("liqi@lly.cn"), Gender: false},
		{ID: 7, Name: "晓梅", Age: 25, Email: PtrString("xiaomeo@sl.com"), Gender: false},
		{ID: 8, Name: "如燕", Age: 26, Email: PtrString("ruyan@yf.com"), Gender: false},
		{ID: 9, Name: "魔灵", Age: 21, Email: PtrString("moling@sl.com"), Gender: true},
	}

	DB.Save(&studentList)//创建或更新记录
}

// PtrString 用于将指针内容传给email
func PtrString(email string) *string {
	return &email
}
```
    表格内容如下：追加9个数据
![Alt text](image.png)

# where查询
sql原句：
```sql
#查询用户名是枫枫的
select * from students where name='枫枫';

#查询用户名不是枫枫的
select * from students where name != '枫枫';

#查询用户名包含 如燕，李元芳的   %匹配多个字符
select * from students where name in ('如燕','李元芳');

#查询姓李的 即李后面有多个字符
select * from students where name like '李%';

#查询年龄大于23，是qq邮箱的
select *from students where age>23 and email like '%qq%';

#查询是qq邮箱的，或者是女的
select *from students where email like '%qq%' or Gender =0;
```

---

go源程序
```GO
package main

import (
	"encoding/json"
	"fmt"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

type Student struct {
	ID     uint    `gorm:"size:10"`
	Name   string  `gorm:"size:16"`
	Age    int     `gorm:"size:3"`
	Email  *string `gorm:"size:128"`
	Gender bool
}

var DB *gorm.DB

func init() {
	username := "root"   //账号
	password := "123456" //密码
	host := "127.0.0.1"  //数据库地址，可以是Ip或者域名
	port := 3306         //数据库端口
	Dbname := "gorm"     //数据库名
	timeout := "10s"     //连接超时，10秒

	// root:root@tcp(127.0.0.1:3306)/gorm?
	dsn := fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8mb4&parseTime=True&loc=Local&timeout=%s", username, password, host, port, Dbname, timeout)
	//连接MYSQL, 获得DB类型实例，用于后面的数据库读写操作。
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info),
	})
	if err != nil {
		panic("连接数据库失败, error=" + err.Error())
	}
	DB = db
	// 连接成功
	fmt.Println(db)
}

func main() {
	//	#查询用户名是枫枫的
	//	select * from students where name='枫枫';
	var SList []Student
	DB.Where("name=?", "枫枫").Find(&SList)
	fmt.Println(SList)

	clean(&SList) //清空切片，但是后面发现每次会重新赋值，不用清空

	//SList = []Student{}

	//	#查询用户名不是枫枫的 NOT
	//	select * from students where name != '枫枫';
	DB.Where("name!=?", "枫枫").Find(&SList)
	fmt.Println(SList)

	//	#查询用户名包含 如燕，李元芳的   %匹配多个字符   IN 通配符
	//	select * from students where name in ('如燕','李元芳');
	//DB.Where("name in (?,?)", "如燕", "李元芳").Find(&SList)
	DB.Where("name in ?", []string{"如燕", "李元芳"}).Find(&SList)
	fmt.Println(SList)

	//	#查询姓李的 即李后面有多个字符 通配符
	//	select * from students where name like '李%';
	DB.Where("name like?", "李%").Find(&SList)
	fmt.Println(SList)

	//	#查询年龄大于23，是qq邮箱的 AND
	//	select *from students where age>23 and email like '%qq%';
	DB.Where("age>? and email like ?", 23, "%qq%").Find(&SList)
	stringPointer(SList)

	//	#查询是qq邮箱的，或者是女的 OR
	//	select *from students where email like '%qq%' or Gender =0;
	DB.Where("email like ? or Gender =?", "%qq%", "0").Find(&SList)
	stringPointer(SList)

	DB.Where(" Gender =?", false).Find(&SList)
	DB.Where(" Gender =?", "false").Find(&SList)
	stringPointer(SList)
	DB.Where(" Gender =?", "0").Find(&SList)
	stringPointer(SList)
}

// 将指针传进去，直接清空切片内容
func clean(s *[]Student) {
	*s = []Student{}
}

// 将email指针内容显示
func stringPointer(s []Student) {
	for _, student := range s {
		marshal, _ := json.Marshal(student)
		fmt.Println(string(marshal))
	}
}

```