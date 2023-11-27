# 一对多关系 Has Many部分
[官方文档](https://gorm.io/zh_CN/docs/has_many.html#Has-Many)
has many 与另一个模型建立了一对多的连接。 不同于 has one，拥有者可以有零或多个关联模型。
>老板与员工
**女神和舔狗**
老师和学生
班级与学生
用户与文章...

&emsp;&emsp;例如，您的应用包含 user 和 article 模型，且每个 user 可以有多篇 article。

# 声明
&emsp;&emsp;user为**主表**，由于一对多，在user结构中增加article切片；
&emsp;&emsp;article表为**副表**，在其中加入外键关联，即主表名+ID，再加入user结构体
```go
// User 用户表 一个用户可以有多篇文章
type User struct {
	ID       uint   `gorm:"size:4"`
	Name     string `gorm:"size:8"`
	Articles []Article
}

// Article 文章表 一篇文章属于一个用户
type Article struct {
	ID     uint   `gorm:"size:4"`
	Title  string `gorm:"size:16"`
	UserID uint   `gorm:"size:4"`
	User   User
}
//使用automigrate进行表的迁移
DB.AutoMigrate(&User{}, &Article{})
```
&emsp;&emsp;**默认的外键名**是拥有者的类型名加上其主键字段名，这里外键名称就是关联表名+ID，不加备注不可以随便更改，会报错；且外键类型要和主表关联的字段类型一直，这里都为unit，size：4
可以得到建立的表的关系如图：

![Alt text](image-1.png)

# 重写外键关联
&emsp;&emsp;想要使用另一个字段作为外键，您可以使用 foreignKey 标签自定义它：
```go
//重写外键关联----------------------------
//gorm的foreignKey备注写在对应的两个表的关联上
type User1 struct {
	ID       uint       `gorm:"size:4"`
	Name     string     `gorm:"size:8"`
	Articles []Article1 `gorm:"foreignKey:UID"`
}

type Article1 struct {
	ID    uint   `gorm:"size:4"`
	Title string `gorm:"size:16"`
	UID   uint   `gorm:"size:4"`
	User  User1  `gorm:"foreignKey:UID"`
}
```
关联外键的数据类型仍需一致，但是字段名不再受限制

# ~~重写引用~~ 

>**这里博主大失败，最后也没有搞出来，希望过路大神解答**
这里贴[官方文档](https://docs.fengfengzhidao.com/#/docs/gorm%E6%96%87%E6%A1%A3/7.%E4%B8%80%E5%AF%B9%E5%A4%9A%E5%85%B3%E7%B3%BB?id=%e9%87%8d%e5%86%99%e5%a4%96%e9%94%ae%e5%bc%95%e7%94%a8)

&emsp;&emsp;GORM 通常使用拥有者的**主键**作为外键的值。 对于上面的例子，它是 User 的 ID 字段。
&emsp;&emsp;为 user 添加 credit card 时，GORM 会将 user 的 ID 字段保存到 credit card 的 UserID 字段。
&emsp;&emsp;同样的，您也可以使用标签 references 来更改它，例如：
```go
//重写引用----------------------------
//备注写在对应的两个表的关联上

type User2 struct {
	ID       uint       `gorm:"size:4"`
	Name     string     `gorm:"size:8"`
	Articles []Article2 `gorm:"foreignKey:UserName;references:Name"`
}

type Article2 struct {
	ID       uint   `gorm:"size:4"`
	Title    string `gorm:"size:16"`
	UserName string `gorm:"size:8"`
	User     User2  `gorm:"references:Name"`
}
```
报错信息
> Error 1170 (42000): BLOB/TEXT column 'company_id' used in key specification without a key length

# 一对多的添加
```go
//创建用户的同时创建文章，并将两者关联
DB.Save(&User{
	Name: "wang2",
	Articles: []Article{
		{Title: "golang"},
		{Title: "python"},
	},
})

//创建文章，关联已有用户
var user User
DB.Take(&user, 1) //查询已有用户
DB.Save(&Article{Title: "c++", User: user})//将关联部分的User结构体传入
```

# 外键添加、查询、嵌套预加载、删除、清楚外键关系。。。马上到来