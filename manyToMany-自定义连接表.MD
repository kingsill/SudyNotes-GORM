[toc]
# 多对多 many2many
Many to Many 会在两个 model 中添加一张连接表，将不同表的id连接起来，也就是说 总共三张表
我们这里以文章和其tag为例，一篇文章可以有多个tag，一个tag中也有多个文章

## 表结构搭建
```go
type Article struct {
	ID    uint   `gorm:"size:8"`
	Title string `gorm:"size:16"`
	Tags  []Tag  `gorm:"many2many:article_tags;"` //用于确定多对多的关系并指定第三张连接表的名字
}
type Tag struct {
	ID       uint      `gorm:"size:8"`
	Text     string    `gorm:"size:16"`
	Articles []Article `gorm:"many2many:article_tags;"`//反向引用，可以用来查询具有相同标签的文章
}

DB.AutoMigrate(&Article{}, &Tag{})
```
搭建表结构如图：
![Alt text](image-3.png)

## 多对多添加
创建标签和文章：
&emsp;&emsp;建立一篇标题为“go”的文章，并新建标签为``study``和``language``。
```go
DB.Save(&Article{
	ID:    1,
	Title: "go",
	Tags: []Tag{
		{Text: "language"},
		{Text: "study"},
	},
})
```
添加文章，选择标签:这里以选择单个标签为例
如果不查询，即使标签名字一致也会重建一个新的标签
```go
var tag Tag
DB.Take(&tag, "Text=?", "study")
DB.Save(&Article{
	Title: "Study notes",
	Tags:  []Tag{tag},
})
```
## 多对多查询

```go
//查询文章，显示文章的标签列表
var article Article
DB.Preload("Tags").Take(&article, 1)
fmt.Println(article)

//查询标签，显示具有该标签的文章列表
var tag Tag
DB.Preload("Articles").Take(&tag, 2)
fmt.Println(tag)
```
## 多对多的删除、更新

移除文章的标签
```go
var article Article
DB.Preload("Tags").Take(&article, 1)
DB.Model(&article).Association("Tags").Delete(article.Tags)
fmt.Println(article)
```
跟新文章的标签
```go
article = Article{}
var tags []Tag

DB.Find(&tags, []int{1, 2, 3}) //找到想要添加的标签

DB.Preload("Tags").Take(&article, 1) //预加载要修改的文章

DB.Model(&article).Association("Tags").Replace(tags) //替换文章标签
```
# 自定义连接表
默认的连接表，只有双方的主键id，展示不了更多信息了，为了
```go
type Article struct {
	ID    uint
	Title string
	Tags  []Tag `gorm:"many2many:article2tag;"`
}

type Tag struct {
	ID   uint
	Text string
	//Articles []Article `gorm:"many2many:article2tag;"` //当使用反向引用时需要在setUpJoinTable时多设置一次这个表的 @@@
}

// Article2tag 自定义连接表
type Article2tag struct {
	ArticleID uint      `gorm:"primaryKey"`
	TagID     uint      `gorm:"primaryKey "` //上两项即为连接表默认项
	CreatedAt time.Time //自定义添加一个创建时间字段
}
```
## 生成表结构
```go
//第一个参数为具有连接另一个表的字段的连接表，第二个即为连接字段的字段名，第三个为连接表
DB.SetupJoinTable(&Article{}, "Tags", &Article2tag{})
//DB.SetupJoinTable(&Tag{}, "Articles", &Article2tag{}) //与上面反向引用时对应 @@@
DB.AutoMigrate(&Article{}, &Tag{}, &Article2tag{})
```

## 操作案例
>&emsp;&emsp;我们如果在自定义连接表中设置了默认字段之外的字段的话，需要使用添加钩子``beforeCreate``来赋值,这里的``createAt``字段gorm会自动填充，关于[钩子函数](https://blog.csdn.net/kingsill/article/details/134607359?spm=1001.2014.3001.5501)
>&emsp;&emsp;**SetupJoinTable**:添加和更新是不能注释掉这个，这样才能走自定义的连接表，以及走他的钩子函数，查询则不需要

### 添加文章并添加标签，并自动关联
```go
DB.Create(&Article{
	Title: "golang_study",
	Tags: []Tag{{Text: "go"},
		{Text: "study"}},
	})
```

### 添加文章，关联已有标签
```go
//添加文章，关联已有标签
var tag Tag
DB.Take(&tag, "text=?", "study")
DB.Create(&Article{
	Title: "how To Study",
	Tags:  []Tag{tag},
})
```

### 给已有文章关联标签
```go
//直接操作连接表，比较极端，知道文章和tag的ID，直接进行添加-------------
DB.Create(&Article2tag{
	ArticleID: 3,
	TagID:     2,
	CreatedAt: time.Time{},
})

//先查询相关标签，在关联--------------------
var tags []Tag //这里主要根据tag切片作为识别，切片名字可以随便起
DB.Find(&tags, "text in?", []string{"go", "study"})

var article Article
DB.Take(&article, "title=?", "gin")

DB.Model(&article).Association("Tags").Append(&tags)
```

### 替换已有文章的标签
```go
//替换已有文章的标签
var article Article
DB.Preload("Tags").Take(&article, 3) //preload中参数需严格对应,这里不需要preload好像也可以：) 
fmt.Println(article)
var tag Tag
DB.Take(&tag, "text=?", "go")
DB.Model(&article).Association("Tags").Replace(&tag)
```

### 查询文章列表，显示标签 <span id="jump"></span>
```go
//查询文章列表并显示标签
var articles []Article
DB.Preload("Tags").Find(&articles)
fmt.Println(articles)
```

## 自定义连接表主键
&emsp;&emsp;当定义的表复杂时，gorm自动的主键名可能比较冗长，我们可以通过自定义主键名来简单化
主要要注释的是``joinForeignKey`` 连接主键ID
&emsp; &emsp; &emsp; &emsp; &emsp; &emsp; ``joinReferences`` 关联主键ID 

```go
// 自定义主键部分-----------------------------------------
type Article struct {
	ID    uint
	Title string
	Tags  []Tag `gorm:"many2many:article2tag;joinForeignKey:A_ID;joinReferences:T_ID"`
}

type Tag struct {
	ID       uint
	Text     string
	Articles []Article `gorm:"many2many:article2tag;joinForeignKey:T_ID;joinReferences:A_ID"` //当使用反向引用时需要在setUpJoinTable时多设置一次这个表的 @@@
}

// Article2tag 自定义连接表
type Article2tag struct {
	A_ID      uint      `gorm:"primaryKey"`
	T_ID      uint      `gorm:"primaryKey "` //上两项即为连接表默认项
	CreatedAt time.Time //自定义添加一个创建时间字段
}
```
## 操作连接表
&emsp;&emsp;当通过其他表来查连接表，可能会不方便，比如我们在这里查二者什么时候建立的关联，即createat字段名
### 查询
[传统查询文章列表，显示标签](#jump)
首先其无法进行分页操作，经过如下修改，也能分页
```go
DB.Preload("Tags").Take(&article, 1)
var tags []Tag
DB.Model(article).Limit(1).Association("Tags").Find(&tags) //这样可以分页，但是也无法查询连接表中其他字段，如关联时间
fmt.Println(tags)
```
若是直接在此时的连接表查询，我们也只能得到文章和标签的id，无法得到用户名及标签名
因此我们可以改一下连接表结构,将两张主表的结构体关联到连接表中，具体可见如下代码所示：
```go
// Article2tag 自定义连接表
type Article2tag struct {
	ArticleID uint    `gorm:"primaryKey"`
	TagID     uint    `gorm:"primaryKey "` //上两项即为连接表默认项
	Article   Article `gorm:"foreignKey:ArticleID"`
	Tag       Tag     `gorm:"foreignKey:TagID"`

	CreatedAt time.Time //自定义添加一个创建时间字段
}
```
查询相关写法可以参见该 [链接](https://blog.csdn.net/kingsill/article/details/134561991?spm=1001.2014.3001.5501#t14)
```go
var articleTag []Article2tag
DB.Preload("Article").Preload("Tag").Where(map[string]any{"Article_id": 1}).Find(&articleTag) //大小写可忽略，符号不可忽略，具体需查看表内字段
fmt.Println(articleTag)
DB.Preload("Article").Preload("Tag").Where("article_id=?", 1).Find(&articleTag) //大小写可忽略，符号不可忽略，具体需查看表内字段
fmt.Println(articleTag)
DB.Preload("Article").Preload("Tag").Find(&articleTag, "article_id=?", 1) //大小写可忽略，符号不可忽略，具体需查看表内字段
fmt.Println(articleTag)
```

