# xorm
xorm是一个简单而强大的Go语言ORM库. 通过它可以使数据库操作非常简便。

## 说明

* 本库是基于原版 <b>xorm</b>：[https://github.com/go-xorm/xorm](https://github.com/go-xorm/xorm) 的定制增强版本，由于本定制版有第三方库依赖（原版xorm无任何第三方库依赖），原版xorm要保持对第三方库零依赖特性，所以只好单独开了本Github库。
* 本库的相关定制功能是为了解决更简单的进行复杂SQL调用和一些特殊业务需求场景而开发的。
* 本定制版ORM相关核心功能和原版保持一致，会跟随原版xorm更新。
* 定制功能采用针对原版弱侵入性代码实现。

## 特性

* 支持Struct和数据库表之间的灵活映射，并支持自动同步
* 事务支持
* 同时支持原始SQL语句和ORM操作的混合执行
* 支持类ibatis方式配置SQL语句（支持xml配置文件和pongo2模板2种方式）
* 支持动态SQL功能
* 支持一次批量混合执行多个CRUD操作，并返回多个结果集
* 使用连写来简化调用
* 支持使用Id, In, Where, Limit, Join, Having, Table, Sql, Cols等函数和结构体等方式作为条件
* 支持级联加载Struct
* 支持数据库查询结果直接返回Json字符串和xml字符串
* 支持缓存
* 支持根据数据库自动生成xorm的结构体
* 支持记录版本（即乐观锁）

## 驱动支持

目前支持的Go数据库驱动和对应的数据库如下：

* Mysql: [github.com/go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)
* MyMysql: [github.com/ziutek/mymysql](https://github.com/ziutek/mymysql)
* Postgres: [github.com/lib/pq](https://github.com/lib/pq)
* Tidb: [github.com/pingcap/tidb](https://github.com/pingcap/tidb)
* SQLite: [github.com/mattn/go-sqlite3](https://github.com/mattn/go-sqlite3)
* MsSql: [github.com/denisenkom/go-mssqldb](https://github.com/denisenkom/go-mssqldb)
* MsSql: [github.com/lunny/godbc](https://github.com/lunny/godbc)
* Oracle: [github.com/mattn/go-oci8](https://github.com/mattn/go-oci8) (试验性支持)
* ql: [github.com/cznic/ql](https://github.com/cznic/ql) (试验性支持)

## 安装

推荐使用 [gopm](https://github.com/gpmgo/gopm) 进行安装：

	gopm get github.com/xormplus/xorm

或者您也可以使用go工具进行安装：

	go get -u github.com/xormplus/xorm

# 快速开始

* 第一步创建引擎，driverName, dataSourceName和database/sql接口相同

```go
//xorm原版标准方式创建引擎
engine, err := xorm.NewEngine(driverName, dataSourceName)

//您也可以针对特定数据库及数据库驱动使用类似下面的快捷方式创建引擎
engine, err = xorm.NewPostgreSQL(dataSourceName)
engine, err = xorm.NewSqlite3(dataSourceName)
```

* 创建引擎，且需要使用类ibatis方式配置SQL语句，参考如下方式

```go
var err error
engine, err = xorm.NewPostgreSQL("postgres://postgres:root@localhost:5432/testdb?sslmode=disable")

if err != nil {
	t.Fatal(err)
}

/*--------------------------------------------------------------------------------------------------
1、使用SetSqlMapRootDir()方法设置SqlMap配置文件总根目录，返回Engine实例本身，如采用sql/xormcfg.ini配置文件中的配置，可直接使用InitSqlMap()初始化
2、使用SetSqlTemplateRootDir()方法设置SqlTemplate模板配置文件总根目录，返回Engine实例本身，如采用sql/xormcfg.ini配置文件中的配置，可直接使用InitSqlTemplate()初始化
3、SqlMap配置文件总根目录和SqlTemplate模板配置文件总根目录，可代码指定，也可在配置文件中配置，代码指定优先级高于配置
--------------------------------------------------------------------------------------------------*/

//初始化SqlMap配置，可选功能，如应用中无需使用SqlMap，可无需初始化
err = engine.SetSqlMapRootDir("./sql/oracle").InitSqlMap()
if err != nil {
	t.Fatal(err)
}
//初始化动态SQL模板配置，可选功能，如应用中无需使用SqlTemplate，可无需初始化
err = engine.SetSqlTemplateRootDir("./sql/oracle").InitSqlTemplate(xorm.SqlTemplateOptions{Extension: ".xx"})
if err != nil {
	t.Fatal(err)
}

//开启SqlMap配置文件和SqlTemplate配置文件更新监控功能，将配置文件更新内容实时更新到内存，如无需要可以不调用该方法
err = engine.StartFSWatcher()
if err != nil {
	t.Fatal(err)
}

```

* <b>db.InitSqlMap()过程</b>
    * 如使用SetSqlMapRootDir()方法指定SqlMap配置文件总根目录，则InitSqlMap()方法按指定目录遍历SqlMapRootDir所配置的目录及其子目录下的所有xml配置文件（<a href="https://github.com/xormplus/xorm/blob/master/test/sql/oracle/studygolang.xml">配置文件样例 </a>）
    * 如未使用SetSqlMapRootDir()方法指定SqlMap配置文件总根目录，则读取程序所在目下的sql/xormcfg.ini配置文件(<a href="https://github.com/xormplus/xorm/blob/master/test/sql/xormcfg.ini">样例</a>)中的SqlMapRootDir配置项，遍历SqlMapRootDir所配置的目录及其子目录下的所有xml配置文件（<a href="https://github.com/xormplus/xorm/blob/master/test/sql/oracle/studygolang.xml">配置文件样例 </a>）
    * 解析所有配置SqlMap的xml配置文件
    * 配置文件中sql标签的id属性值作为SqlMap的key，如有重名id，则后加载的覆盖之前加载的配置sql条目
    * 配置文件中sql配置会读入内存并缓存
    * 由于SqlTemplate模板能完成更多复杂组装和特殊场景需求等强大功能，故SqlMap的xml只提供这种极简配置方式，非ibatis的OGNL的表达式实现方式

* <b>db.InitSqlTemplate()过程</b>
    * 如使用SetSqlTemplateRootDir()方法指定SqlTemplate模板配置文件总根目录，则InitSqlTemplate()方法按指定目录遍历SqlTemplateRootDir所配置的目录及其子目录及其子目录下的所有stpl模板文件（<a href="https://github.com/xormplus/xorm/blob/master/test/sql/oracle/select.example.stpl">模板文件样例</a>）
    * 如指使用SetSqlTemplateRootDir()方法指定SqlTemplate模板配置文件总根目录，则InitSqlTemplate()方法读取程序所在目下的sql/xormcfg.ini配置文件(<a href="https://github.com/xormplus/xorm/blob/master/test/sql/xormcfg.ini">样例</a>)中的SqlTemplateRootDir配置项，遍历SqlTemplateRootDir所配置的目录及其子目录下的所有stpl模板文件（<a href="https://github.com/xormplus/xorm/blob/master/test/sql/oracle/select.example.stpl">模板文件样例</a>）
    * 解析stpl模板文件
    * stpl模板文件名作为SqlTemplate存储的key（不包含目录路径），如有不同路径下出现同名文件，则后加载的覆盖之前加载的配置模板内容
    * stpl模板内容会读入内存并缓存

* 支持最原始的SQL语句查询

```go
/*-------------------------------------------------------------------------------------
 * 第1种方式：返回的结果类型为 []map[string][]byte
-------------------------------------------------------------------------------------*/
sql_1 := "select * from user"
results, err := engine.Query(sql_1)

/*-------------------------------------------------------------------------------------
 * 第2种方式：返回的结果类型为 []map[string]interface{}
-------------------------------------------------------------------------------------*/
sql_2_1 := "select * from user"
results, err := engine.Sql(sql_2_1).Query().List()
//当然也支持这种方法，将数据库中的时间字段格式化，时间字段对应的golang数据类型为time.Time
//当然你也可以在数据库中先使用函数将时间类型的字段格式化成字符串，这里只是提供另外一种方式
//该方式会将所有时间类型的字段都格式化，所以请依据您的实际需求按需使用
results, err := engine.Sql(sql_2_1).QueryWithDateFormat("20060102").List()

sql_2_2 := "select * from user where id = ? and age = ?"
results, err := engine.Sql(sql_2_2, 7, 17).Query().List()

//此Query()方法返回对象还支持ListPage()方法和Count()方法，这两个方法都是针对数据库查询出来后的结果集进行操作
//此Query()方法返回对象还支持Xml()方法、XmlIndent()方法和Json()方法，相关内容请阅读之后的章节
//ListPage()方法并非数据库分页方法，只是针对数据库查询出来后的结果集[]map[string]interface{}对象取部分切片
//例如以下例子，是取结果集的第1条到第50条记录
results, err := engine.Sql(sql_2_2, 7, 17).Query().ListPage(1,50)
//例如以下例子，是取结果集的第13条到第28条记录
results, err := engine.Sql(sql_2_2, 7, 17).Query().ListPage(13,28)
//此Count()方法也并非使用数据库count函数查询数据库某条件下的记录数，只是针对Sql语句对数据库查询出来后的结果集[]map[string]interface{}对象的数量
//此Count()方法也并非Engine对象和Session对象下的Count()方法，使用时请区分场景
count, err := engine.Sql(sql_2_2, 7, 17).Query().Count()

/*-------------------------------------------------------------------------------------
  第3种方式：执行SqlMap配置文件中的Sql语句，返回的结果类型为 []map[string]interface{}
-------------------------------------------------------------------------------------*/
sql_id_3_1 := "sql_3_1" //配置文件中sql标签的id属性,SqlMap的key
results, err := engine.SqlMapClient(sql_3_1).Query().List()

sql_id_3_2 := "sql_3_2"
results, err := engine.SqlMapClient(sql_id_3_2, 7, 17).Query().List()

sql_id_3_3 := "sql_3_3"
paramMap_3_3 := map[string]interface{}{"id": 7, "name": "xormplus"}
results1, err := engine.SqlMapClient(sql_id_3_3, &paramMap_3_3).Query().List()

/*-------------------------------------------------------------------------------------
 * 第4种方式：执行SqlTemplate配置文件中的Sql语句，返回的结果类型为 []map[string]interface{}
-------------------------------------------------------------------------------------*/
sql_key_4_1 := "select.example.stpl" //配置文件名,SqlTemplate的key

//执行的 sql：select * from user where id=7
//如部分参数未使用，请记得使用对应类型0值，如此处name参数值为空字符串，模板使用指南请详见pongo2
paramMap_4_1 := map[string]interface{}{"count": 1, "id": 7, "name": ""}
results, err := engine.SqlTemplateClient(sql_key_4_1, &paramMap_4_1).Query().List()

//执行的 sql：select * from user where name='xormplus'
//如部分参数未使用，请记得使用对应类型0值，如此处id参数值为0，模板使用指南请详见pongo2
paramMap_4_2 := map[string]interface{}{"id": 0, "count": 2, "name": "xormplus"}
results, err := engine.SqlTemplateClient(sql_key_4_1, &paramMap_4_2).Query().List()

/*-------------------------------------------------------------------------------------
 * 第5种方式：返回的结果类型为对应的[]interface{}
-------------------------------------------------------------------------------------*/
var categories []Category
err := engine.Sql("select * from category where id =?", 16).Find(&categories)

/*-------------------------------------------------------------------------------------
 * 第6种方式：返回的结果类型为对应的[]interface{}
-------------------------------------------------------------------------------------*/
sql_id_6_1 := "sql_6_1"
var categories []Category
err := engine.SqlMapClient(sql_id_6_1, 16).Find(&categories)

sql_id_6_2 := "sql_6_2"
var categories []Category
paramMap_6_2 := map[string]interface{}{"id": 25}
err := engine.SqlMapClient(sql_id_6_2, &paramMap_6_2).Find(&categories)

/*-------------------------------------------------------------------------------------
 * 第7种方式：返回的结果类型为对应的[]interface{}
-------------------------------------------------------------------------------------*/
//执行的 sql：select * from user where name='xormplus'
sql_key_7_1 := "select.example.stpl" //配置文件名,SqlTemplate的key
var users []User
paramMap_7_1 := map[string]interface{}{"id": 0, "count": 2, "name": "xormplus"}
err := engine.SqlTemplateClient(sql_key_7_1, &paramMap_7_1).Find(&users)
```

* 注：
	* 除以上7种方式外，本库还支持另外3种方式，由于这3种方式支持一次性批量混合CRUD操作，返回多个结果集，且支持多种参数组合形式，内容较多，场景比较复杂，因此不在此处赘述。
	* 欲了解另外3种方式相关内容您可移步[批量SQL操作](#ROP_ARM)章节，此3种方式将在此章节单独说明

* 采用Sql(),SqlMapClient(),SqlTemplateClient()方法执行sql调用Find()方法，与ORM方式调用Find()方法不同（传送门：[ORM方式操作数据库](#ORM)），此时Find()方法中的参数，即结构体的名字不需要与数据库表的名字映射（因为前面的Sql()方法已经确定了SQL语句），但字段名需要和数据库中的字段名字做映射。使用Find()方法需要自己定义查询返回结果集的结构体，如不想自己定义结构体可以使用Query()方法，返回[]map[string]interface{}，两种方式请依据实际需要选用。

	举例：多表联合查询例子如下

```go
//执行的SQL如下，查询的是article表与category表，查询的表字段是这两个表的部分字段
sql := `SELECT
	article.id,
	article.title,
	article.isdraft,
	article.lastupdatetime,
	category.name as categoryname
FROM
	article,
	category
WHERE
	article.categorysubid = category. ID
AND category. ID = 4`

//我们可以定义一个结构体,注意：结构体中的字段名和上面执行的SQL语句字段名映射，字段数据类型正确
//当然你还可以给这个结构体加更多其他字段，但是如果执行上面的SQL语句时，这些其他字段只会被赋值对应数据类型的零值
type CategoryInfo struct {
	Id             int     
	Title          string    
	Categoryname   string    
	Isdraft        int       
	Lastupdatetime time.Time 
}

var categoryinfo []CategoryInfo

//执行sql，返回值为error对象，同时查询的结果集会被赋值给[]CategoryInfo
err = db.Sql(sql).Find(&categoryinfo)
if err != nil {
	t.Fatal(err)
}
t.Log(categoryinfo)
t.Log(categoryinfo[0].Categoryname)
t.Log(categoryinfo[0].Id)
t.Log(categoryinfo[0].Title)
t.Log(categoryinfo[0].Isdraft)
t.Log(categoryinfo[0].Lastupdatetime)

```


* 第3种和第6种方式所使用的SqlMap配置文件内容如下

```xml
<sqlMap>
	<sql id="sql_3_1">
		select * from user
	</sql>
	<sql id="sql_3_2">
		select * from user where id=? and age=?
	</sql>
    <sql id="sql_3_3">
		select * from user where id=?id and name=?name
	</sql>
    <sql id="sql_id_6_1">
		select * from category where id =?
	</sql>
    <sql id="sql_id_6_2">
		select * from category where id =?id
	</sql>
</sqlMap>
```

* 第4种和第7种方式所使用的SqlTemplate配置文件内容如下，文件名：select.example.stpl，路径为engine.SqlMap.SqlMapRootDir配置目录下的任意子目录中。使用模板方式配置Sql较为灵活，可以使用pongo2引擎的相关功能灵活组织Sql语句以及动态SQL拼装。

```java
select * from user
where
{% if count>1%}
id=?id
{% else%}
name=?name
{% endif %}
```

* 执行一个SQL语句

```go
//第1种方式
affected, err := engine.Exec("update user set age = ? where name = ?", age, name)

//第2种方式
sql_2 := "INSERT INTO config(key,value) VALUES (?, ?)"
affected, err := engine.Sql(sql_4, "OSCHINA", "OSCHINA").Execute()

//第3种方式
sql_i_1 := "sql_i_1" //SqlMap中key为 "sql_i_1" 配置的Sql语句为：INSERT INTO config(key,value) VALUES (?, ?)
affected, err := engine.SqlMapClient(sql_i_1, "config_1", "1").Execute()

sql_i_2 := "sql_i_2" //SqlMap中key为 "sql_i_2" 配置的Sql语句为：INSERT INTO config(key,value) VALUES (?key, ?value)
paramMap_i := map[string]interface{}{"key": "config_2", "value": "2"}
affected, err := engine.SqlMapClient(sql_i_2, &paramMap_i).Execute()

//第4种方式
sql_i_3 := "insert.example.stpl"
paramMap_i_t := map[string]interface{}{"key": "config_3", "value": "3"}
affected, err := engine.SqlTemplateClient(sql_i_3, &paramMap_i_t).Execute()
```

* 注：
	* 除以上4种方式外，本库还支持另外3种方式，由于这3种方式支持一次性批量混合CRUD操作，返回多个结果集，且支持多种参数组合形式，内容较多，场景比较复杂，因此不在此处赘述。
	* 欲了解另外3种方式相关内容您可移步[批量SQL操作](#ROP_ARM)章节，此4种方式将在此章节单独说明

* 支持链式读取数据操作查询返回json或xml字符串

```go
//第1种方式
var users []User
results,err := engine.Where("id=?", 6).Search(&users).Xml() //返回查询结果的xml字符串
results,err := engine.Where("id=?", 6).Search(&users).Json() //返回查询结果的json字符串

//第2种方式
sql := "select * from user where id = ?"
results, err := engine.Sql(sql, 2).Query().Json() //返回查询结果的json字符串
results, err := engine.Sql(sql, 2).QueryWithDateFormat("20060102").Json() //返回查询结果的json字符串，并支持格式化日期
results, err := engine.Sql(sql, 2).QueryWithDateFormat("20060102").Xml() //返回查询结果的xml字符串，并支持格式化日期

sql := "select * from user where id = ?id and userid=?userid"
paramMap := map[string]interface{}{"id": 6, "userid": 1} //支持参数使用map存放
results, err := engine.Sql(sql, &paramMap).Query().XmlIndent("", "  ", "article") //返回查询结果格式化后的xml字符串

//第3种方式
sql_id_3_1 := "sql_3_1" //配置文件中sql标签的id属性,SqlMap的key
results, err := engine.SqlMapClient(sql_id_3_1, 7, 17).Query().Json() //返回查询结果的json字符串

sql_id_3_2 := "sql_3_2" //配置文件中sql标签的id属性,SqlMap的key
paramMap := map[string]interface{}{"id": 6, "userid": 1} //支持参数使用map存放
results, err := engine.SqlMapClient(sql_id_3_2, &paramMap).Query().Xml() //返回查询结果的xml字符串

//第4种方式
sql_key_4_1 := "select.example.stpl"
paramMap_4_1 := map[string]interface{}{"id": 6, "userid": 1}
results, err := engine.SqlTemplateClient(sql_key_4_1, &paramMap_4_1).Query().Json()
```

* 支持链式读取数据操作查询返回某条记录的某个字段的值

```go
//第1种方式
id := engine.Sql(sql, 2).Query().Results[0]["id"] //返回查询结果的第一条数据的id列的值

//第2种方式
id := engine.SqlMapClient(key, 2).Query().Results[0]["id"] //返回查询结果的第一条数据的id列的值
id := engine.SqlMapClient(key, &paramMap).Query().Results[0]["id"] //返回查询结果的第一条数据的id列的值

//第3种方式
id := engine.SqlTemplateClient(key, &paramMap).Query().Results[0]["id"] //返回查询结果的第一条数据的id列的值
```

* 事务处理，当使用事务处理时，需要创建Session对象，另外当使用Sql()、SqlMapClient()、SqlTemplateClient()方法进行操作时也推荐手工创建Session对象方式管理Session。在进行事物处理时，可以混用ORM方法和RAW方法。注意如果您使用的是mysql，数据库引擎为innodb事务才有效，myisam引擎是不支持事务的。示例代码如下：

```go
session := engine.NewSession()
defer session.Close()
// add Begin() before any action
err := session.Begin()
user1 := Userinfo{Username: "xiaoxiao", Departname: "dev", Alias: "lunny", Created: time.Now()}
_, err = session.Insert(&user1)
if err != nil {
    session.Rollback()
    return
}
user2 := Userinfo{Username: "yyy"}
_, err = session.Where("id = ?", 2).Update(&user2)
if err != nil {
    session.Rollback()
    return
}

_, err = session.Exec("delete from userinfo where username = ?", user2.Username)
if err != nil {
    session.Rollback()
    return
}

_, err = session.SqlMapClient("delete.userinfo", user2.Username).Execute()
if err != nil {
    session.Rollback()
    return
}

// add Commit() after all actions
err = session.Commit()
if err != nil {
    return
}
```

* SqlMap及SqlTemplate相关功能API

```go
//设置SqlMap文件总根目录，可代码指定，也可在配置文件中配置，如使用配置文件中的配置则无需调用该方法，代码指定优先级高于配置
engine.SetSqlMapRootDir()
//设置SqlTemplate模板配置文件总根目录，可代码指定，也可在配置文件中配置，如使用配置文件中的配置则无需调用该方法，代码指定优先级高于配置
engine.SetSqlTemplateRootDir()

err := engine.InitSqlMap()//初始化加载SqlMap配置文件，默认初始化后缀为".xml"，初始化容量100
err := engine.InitSqlTemplate()//初始化加载SqlTemplate配置文件，默认初始化后缀为".stpl"，初始化容量100

//SqlMap配置文件和SqlTemplate配置文件后缀不要相同
//指定SqlMap配置文件后缀为".xx"，但配置内容必须为样例的xml格式，如不指定，默认后缀为".xml"
//同时还可以指定初始化容量，如不指定，默认初始化容量100
option := xorm.SqlMapOptions{Extension: ".xx"}
err := engine.InitSqlMap(option) //按指定SqlMap配置文件后缀为".xx"初始化

//指定SqlTemplate配置文件后缀为".yy"，初始化容量200，如不指定，默认后缀为".stpl"，初始化容量100
option := xorm.SqlTemplateOptions{Extension: ".yy", Capacity: 100}
err = engine.InitSqlTemplate(option) //按指定SqlTemplate配置文件后缀为".yy"初始化

//开启SqlMap配置文件和SqlTemplate配置文件更新监控功能，将配置文件更新内容实时更新到内存，如无需要可以不调用该方法
//该监控模式下，如删除配置文件，内存中不会删除相关配置
engine.StartFSWatcher()
//停止SqlMap配置文件和SqlTemplate配置文件更新监控功能
engine.StopFSWatcher()

/*------------------------------------------------------------------------------------
1、以下方法是在没有engine.InitSqlMap()和engine.InitSqlTemplate()初始化相关配置文件的情况下让您在代码中可以轻松的手动管理SqlMap配置及SqlTemplate模板。
2、engine.InitSqlMap()和engine.InitSqlTemplate()初始化相关配置文件之后也可以使用以下方法灵活的对SqlMap配置及SqlTemplate模板进行管理
3、方便支持您系统中其他初始化配置源，可不依赖于本库的初始化配置方式
4、可在代码中依据业务场景，动态的添加、更新、删除SqlMap配置及SqlTemplate模板
5、手工管理的SqlMap配置及SqlTemplate模板，与xorm初始化方法一样会将相关配置缓存，但不会生成相关配置文件
-----------------------------------------------------------------------------------*/
engine.LoadSqlMap(filepath) //加载指定文件的SqlMap配置
engine.ReloadSqlMap(filepath) //重新加载指定文件的SqlMap配置

engine.BatchLoadSqlMap([]filepath) //批量加载SqlMap配置
engine.BatchReloadSqlMap([]filepath) //批量加载SqlMap配置

engine.GetSql(key, sql) //获取一条SqlMap配置
engine.AddSql(key, sql) //新增一条SqlMap配置
engine.UpdateSql(key, sql) //更新一条SqlMap配置
engine.RemoveSql(key) //删除一条SqlMap配置

engine.BatchAddSql(map[key]sql) //批量新增SqlMap配置
engine.BatchUpdateSql(map[key]sql) //批量更新SqlMap配置
engine.BatchRemoveSql([]key) //批量删除SqlMap配置

engine.LoadSqlTemplate(filepath) //加载指定文件的SqlTemplate模板
engine.ReloadSqlTemplate(filepath) //重新加载指定文件的SqlTemplate模板

engine.BatchLoadSqlTemplate([]filepath) //批量加载SqlTemplate模板
engine.BatchReloadSqlTemplate([]filepath) //批量加载SqlTemplate模板

engine.AddSqlTemplate(key, sql) //新增一条SqlTemplate模板，sql为SqlTemplate模板内容字符串
engine.UpdateSqlTemplate(key, sql) //更新一条SqlTemplate模板，sql为SqlTemplate模板内容字符串
engine.RemoveSqlTemplate(key) //删除一条SqlTemplate模板

engine.BatchAddSqlTemplate(map[key]sql) //批量新增SqlTemplate配置，sql为SqlTemplate模板内容字符串
engine.BatchUpdateSqlTemplate(map[key]sql) //批量更新SqlTemplate配置，sql为SqlTemplate模板内容字符串
engine.batchUpdateSqlTemplate([]key) //批量删除SqlTemplate配置

/*
1、指定多个key，批量查询SqlMap配置,...key的数据类型为...interface{},返回类型为map[string]string
2、支持如下多种调用方式
	a)engine.GetSqlMap("Test_GetSqlMap_1"),返回key为Test_GetSqlMap_1的SqlMap配置
    b)engine.GetSqlMap("Test_GetSqlMap_1", "Test_GetSqlMap_3"),返回key为Test_GetSqlMap_1,Test_GetSqlMap_3的SqlMap配置
    c)engine.GetSqlMap("Test_GetSqlMap_1", "Test_GetSqlMap_3","Test_GetSqlMap_null"),返回key为Test_GetSqlMap_1,Test_GetSqlMap_3的SqlMap，Test_GetSqlMap_null配置，其中Test_GetSqlMap_null在内存中缓存的的key不存在，则在返回的map[string]string中，key Test_GetSqlMap_null配置返回的值为空字符串
    d)engine.GetSqlMap([]string{"Test_GetSqlMap_1", "Test_GetSqlMap_3"})支持字符串数组形式参数
    e)engine.GetSqlMap([]string{"Test_GetSqlMap_1", "Test_GetSqlMap_3"},"Test_GetSqlMap_2")支持字符串数组形式和字符串参数混用
    f)engine.GetSqlMap([]string{"Test_GetSqlMap_1", "Test_GetSqlMap_3"},"Test_GetSqlMap_2"，3)支持字符串数组形式，字符串参数和其他类型参数混用，但查询时只会处理字符串类型参数和字符转数组类型参数（因为SqlMap的key是字符串类型），返回的map[string]string也无其他类型的key
3、如不传任何参数，调用engine.GetSqlMap()，则返回整个内存中当前缓存的所有SqlMap配置
*/
engine.GetSqlMap(...key)

/*
1、指定多个key，批量查询SqlTemplate配置,...key的数据类型为...interface{},返回类型为map[string]*pongo2.Template
2、支持如下多种调用方式
	a)engine.GetSqlTemplates("Test_GetSqlTemplates_1"),返回key为Test_GetSqlTemplates_1的SSqlTemplate配置
    b)engine.GetSqlTemplates("Test_GetSqlTemplates_1", "Test_GetSqlTemplates_3"),返回key为Test_GetSqlTemplates_1,Test_GetSqlTemplates_3的SqlTemplate配置
    c)engine.GetSqlTemplates("Test_GetSqlTemplates_1", "Test_GetSqlTemplates_3","Test_GetSqlTemplates_null"),返回key为Test_GetSqlTemplates_1,Test_GetSqlTemplates_3的SqlMap，Test_GetSqlMap_null配置，其中Test_GetSqlTemplates_null在内存中缓存的的key不存在，则在返回的map[string]*pongo2.Template中，key Test_GetSqlTemplates_null配置返回的值为nil
    d)engine.GetSqlTemplates([]string{"Test_GetSqlTemplates_1", "Test_GetSqlTemplates_3"})支持字符串数组形式参数
    e)engine.GetSqlTemplates([]string{"Test_GetSqlTemplates_1", "Test_GetSqlTemplates_3"},"Test_GetSqlTemplates_2")支持字符串数组形式和字符串参数混用
    f)engine.GetSqlTemplates([]string{"Test_GetSqlTemplates_1", "Test_GetSqlTemplates_3"},"Test_GetSqlTemplates_2"，3)支持字符串数组形式，字符串参数和其他类型参数混用，但查询时只会处理字符串类型参数和字符转数组类型参数（因为SqlTemplate的key是字符串类型），返回的map[string]*pongo2.Template也无其他类型的key
3、如不传任何参数，调用engine.GetSqlTemplates()，则返回整个内存中当前缓存的所有SqlTemplate配置
4、engine.GetSqlTemplates()返回类型为map[string]*pongo2.Template，可以方便的实现链式调用pongo2的Execute()，ExecuteBytes()，ExecuteWriter()方法
*/
engine.GetSqlTemplates(...key)
```
<a name="ROP_ARM"/>
# 批量SQL操作
* 批量SQL操作API

```Go
//第一种方式，可以从Engine对象轻松进行使用，该方式自动管理事务，注意如果您使用的是mysql，数据库引擎为innodb事务才有效，myisam引擎是不支持事务的。
engine.Sqls(sqls, parmas...).Execute()
engine.SqlMapsClient(sqlkeys, parmas...).Execute()
engine.SqlTemplatesClient(sqlkeys, parmas...).Execute()

//第2种方式，手动创建Session对象进行调用，该方式需要您手动管理事务
session := engine.NewSession()
defer session.Close()
// add Begin() before any action
err := session.Begin()


_, err = session.Exec("delete from userinfo where username = ?", user2.Username)
if err != nil {
    session.Rollback()
    return
}

//Execuet返回值有3个，分别为slice,map,error类型
results, _, err = session.Sqls(sqls, parmas...).Execute()
if err != nil {
    session.Rollback()
    return
}

_, results, err = session.SqlMapsClient(sqlkeys, parmas...).Execute()
if err != nil {
    session.Rollback()
    return
}

results, _, err = session.SqlTemplatesClient(sqlkeys, parmas...).Execute()
if err != nil {
    session.Rollback()
    return
}

// add Commit() after all actions
err = session.Commit()
if err != nil {
    return
}

//支持两种返回结果集
//Slice形式类似如下
/*
[
    [
        {
            "id": "6",
            "name": "xorm"
        },
        {
            "id": "7",
            "name": "xormplus"
        },
        {
            "id": "8",
            "name": "ibatis"
        }
    ],
    [
        {
            "LastInsertId": 0,
            "RowsAffected": 0
        }
    ],
    [
        {
            "LastInsertId": 0,
            "RowsAffected": 0
        }
    ],
    [
        {
            "LastInsertId": 13,
            "RowsAffected": 1
        }
    ]
]
 */

//Map形式类似如下
/*
{
    "deleteUser": [
        {
            "LastInsertId": 0,
            "RowsAffected": 0
        }
    ],
    "insertUser": [
        {
            "LastInsertId": 11,
            "RowsAffected": 1
        }
    ],
    "updateUser": [
        {
            "LastInsertId": 0,
            "RowsAffected": 0
        }
    ],
    "userList": [
        {
            "id": "3",
            "name": "xorm"
        },
        {
            "id": "4",
            "name": "xormplus"
        },
    ]
}
 */

```

* Sqls(sqls, parmas...)方法说明：
  1. sqls参数
    * sqls参数数据类型为interface{}类型，但实际参数类型检查时，只支持string，[]string和map[string]string三中类型，使用其他类型均会返回参数类型错误。
    * 使用string类型则为执行单条Sql执行单元（传送门：[Sql执行单元定义](#SQL)），Execute()方法返回的结果集数据类型为[][]map[string]interface{}，只有1个元素。
    * 使用[]string则Execute()方法为有序执行多条Sql执行单元，Execute()方法返回的结果集数据类型为[][]map[string]interface{}类型，结果集元素个数与sqls参数的元素个数相同，每个元素索引与返回结果集的索引一一对应。
    * 使用map[string]string类型则Execute()方法为无序执行多条Sql执行单元，Execute()方法返回的结果集数据类型为map[string][]map[string]interface{}，结果集map的key与返回结果集的key一一对应。
  2. parmas...参数
	* 可以接收0个参数或则1个参数，当所有执行单元都无需执行参数时候，可以不传此参数
	* parmas参数数据类型为interface{}，但实际参数类型检查时，只支持map[string]interface{}，[]map[string]interface{}和map[string]map[string]interface{}三种类型，使用其他类型均会返回参数类型错误。
	* 使用map[string]interface{}类型时候，sqls参数类型必须为string类型，即map[string]interface{}类型为单条Sql执行单元的参数。
	* 使用[]map[string]interface{}类型时候，sqls参数类型可以为string类型，此时只有第一个元素[0]map[string]interface{}会被提取，之后的元素将不起任何作用。同时，sqls参数类型也可以为[]string类型，这种参数组合是最常用的组合形式之一，sqls参数的索引和parmas参数的索引一一对应。当某个索引所对应的Sql执行单元是无参数的时候，请将此索引的值设为nil，即parmas[i] = nil
	* 使用map[string]map[string]interface{}类型时，sqls参数类型必须为map[string]string类型，这种参数组合是最常用的组合形式之一，sqls参数的key和parmas参数的key一一对应。当某个key所对应的Sql执行单元是无参数的时候，请将此key的值设为nil，即parmas[key] = nil

* SqlMapsClient(sqlkeys, parmas...)方法说明：
  1. sqlkeys参数
    * sqlkeys参数数据类型为interface{}类型，但实际参数类型检查时，只支持string，[]string和map[string]string三中类型，使用其他类型均会返回参数类型错误。
    * 使用string类型则为执行单条Sql执行单元（Sql执行单元定义），即在xorm种缓存的SqlMap中的key所对应的配置项，Execute()方法返回的结果集数据类型为[][]map[string]interface{}，只有1个元素。
    * 使用[]string则Execute()方法为有序执行多条Sql执行单元，Execute()方法返回的结果集数据类型为[][]map[string]interface{}类型，结果集元素个数与sqls参数的元素个数相同，sqlkeys的索引与返回结果集的索引一一对应，sqlkeys存储的是每个元素的值是xorm缓存的SqlMap的key
    * 使用map[string]string类型则Execute()方法为无序执行多条Sql执行单元，Execute()方法返回的结果集数据类型为map[string][]map[string]interface{}，sqlkeys的key与返回结果集的key一一对应，sqlkeys存储的是每个键值对的值是xorm缓存的SqlMap的key
  2. parmas...参数
	* 可以接收0个参数或则1个参数，当所有执行单元都无需执行参数时候，可以不传此参数
	* parmas参数数据类型为interface{}，但实际参数类型检查时，只支持map[string]interface{}，[]map[string]interface{}和map[string]map[string]interface{}三种类型，使用其他类型均会返回参数类型错误。
	* 使用map[string]interface{}类型时候，sqlkeys参数类型必须为string类型，即map[string]interface{}类型为单条Sql执行单元的参数。效果等同SqlMapClient()方法（请注意本方法名为SqlMapsClient）。
	* 使用[]map[string]interface{}类型时候，sqlkeys参数类型支持两种：
		* 第1种为string类型，此时只有第一个元素[0]map[string]interface{}会被提取，之后的元素将不起任何作用。
		* 第2种为[]string类型，这种参数组合是最常用的组合形式之一，sqlkeys参数的索引和parmas参数的索引一一对应。当某个索引所对应的Sql执行单元是无参数的时候，请将此索引的值设为nil，即parmas[i] = nil
	* 使用map[string]map[string]interface{}类型时，sqlkeys参数类型必须为map[string]string类型，这种参数组合是最常用的组合形式之一，sqlkeys参数的key和parmas参数的key一一对应。当某个key所对应的Sql执行单元是无参数的时候，请将此key的值设为nil，即parmas[key] = nil

* SqlTemplatesClient(sqlkeys, parmas...)方法说明：
  1. sqlkeys参数
    * sqlkeys参数数据类型为interface{}类型，但实际参数类型检查时，只支持string，[]string和map[string]string三中类型，使用其他类型均会返回参数类型错误。
    * 使用string类型则为执行单条Sql执行单元（Sql执行单元定义），即在xorm缓存的SqlTemplate中的key所对应的模板，Execute()方法返回的结果集数据类型为[][]map[string]interface{}，只有1个元素。
    * 使用[]string则Execute()方法为有序执行多条Sql执行单元，Execute()方法返回的结果集数据类型为[][]map[string]interface{}类型，结果集元素个数与sqls参数的元素个数相同，sqlkeys的索引与返回结果集的索引一一对应，sqlkeys存储的是每个元素的值是xorm缓存的SqlTemplate的key
    * 使用map[string]string类型则Execute()方法为无序执行多条Sql执行单元，Execute()方法返回的结果集数据类型为map[string][]map[string]interface{}，sqlkeys的key与返回结果集的key一一对应，sqlkeys存储的是每个键值对的值是xorm缓存的SqlTemplate的key
  2. parmas...参数
	* 可以接收0个参数或则1个参数，当所有执行单元都无需执行参数时候，可以不传此参数
	* parmas参数数据类型为interface{}，但实际参数类型检查时，只支持map[string]interface{}，[]map[string]interface{}和map[string]map[string]interface{}三种类型，使用其他类型均会返回参数类型错误。
	* 使用map[string]interface{}类型时候，sqlkeys参数类型必须为string类型，即map[string]interface{}类型为单条Sql执行单元的参数。效果等同SqlMapClient()方法（请注意本方法名为SqlMapsClient）。
	* 使用[]map[string]interface{}类型时候，sqlkeys参数类型支持两种：
		* 第1种为string类型，此时只有第一个元素[0]map[string]interface{}会被提取，之后的元素将不起任何作用；
		* 第2种为[]string类型，这种参数组合是最常用的组合形式之一，sqlkeys参数的索引和parmas参数的索引一一对应。当某个索引所对应的Sql执行单元是无参数的时候，请将此索引的值设为nil，即parmas[i] = nil
	* 使用map[string]map[string]interface{}类型时，sqlkeys参数类型必须为map[string]string类型，这种参数组合是最常用的组合形式之一，sqlkeys参数的key和parmas参数的key一一对应。当某个key所对应的Sql执行单元是无参数的时候，请将此key的值设为nil，即parmas[key] = nil

* Execute()方法说明：
	* 一共3个返回值，([][]map[string]interface{}, map[string][]map[string]interface{}, error)
	* 当以上3个方法的sqls或sqlkeys参数为string或[]string时为有序执行Sql执行单元，故返回结果集为第一个返回值，Slice存储，第二返回值为nil
	* 当以上3个方法的sqls或sqlkeys参数为map[string]string时为无序执行Sql执行单元，返回结果集为第二个返回值，map存储，第一个返回值为nil
	* 当以上3个方法执行中出现错误，则第三个返回值有值，前2个返回值均为nil

<a name="SQL"></a>
* Sql执行单元定义
	* 当sqls为string时候，则Sql执行单元为该字符串的内容
	* 当sqlkeys为string时，则Sql执行单元为所对应的SqlMap配置项或SqlTemplate模板
	* 当sqls为[]string或map[string]string时候，则Sql执行单元为相关元素的字符串内容
	* 当sqlkeys为[]string或map[string]string时候，则Sql执行单元为以相关元素为key所对应的SqlMap配置项或SqlTemplate模板
	* Sql执行单元的具体内容，必须以"select", "insert", "delete", "update", "create", "drop"为起始内容,但后续内容不会继续做检查，请合理定义Sql执行单元内容。当执行单元内容不是以上起始内容，则对应索引或key返回的结果集为nil，请注意对返回结果集的nil判断
	* Sql执行单元并非单条Sql语句，当执行insert，delete，update，create，drop操作时候，可以为多条Sql语句，这里需要对应的数据库的SQL语法支持。如在一个执行单元批量执行多条Sql，返回结果集作为一组所有执行单元的大结果集中的一个元素，这个结果集的数据类型为map[string]interface{}，只有2个键值对，一个键值对的key为LastInsertId，一个键值对的key为RowsAffected，请控制好执行粒度。另外，目前不是所有数据库都支持返回LastInsertId，目前还在设计更通用的API来实现所有数据库都能支持此功能。
	* 当执行select操作时候，执行单元的Sql语句必须为一条，返回结果集作为一组所有执行单元的大结果集中的一个元素
	* insert，delete，update，create，drop操作不能和select操作混合定义在同一个执行单元中
	* 最后，Sql执行单元基于以上约定，请合理组织

<a name="ORM"></a>
# ORM方式操作数据库

* 定义一个和表同步的结构体，并且自动同步结构体到数据库

```Go
type User struct {
    Id int64
    Name string
    Salt string
    Age int
    Passwd string `xorm:"varchar(200)"`
    Created time.Time `xorm:"created"`
    Updated time.Time `xorm:"updated"`
}

err := engine.Sync2(new(User))
```

* ORM方式插入一条或者多条记录

```Go
affected, err := engine.Insert(&user)
// INSERT INTO struct () values ()
affected, err := engine.Insert(&user1, &user2)
// INSERT INTO struct1 () values ()
// INSERT INTO struct2 () values ()
affected, err := engine.Insert(&users)
// INSERT INTO struct () values (),(),()
affected, err := engine.Insert(&user1, &users)
// INSERT INTO struct1 () values ()
// INSERT INTO struct2 () values (),(),()
```

* ORM方式查询单条记录

```Go
has, err := engine.Get(&user)
// SELECT * FROM user LIMIT 1
has, err := engine.Where("name = ?", name).Desc("id").Get(&user)
// SELECT * FROM user WHERE name = ? ORDER BY id DESC LIMIT 1
```

* ORM方式查询多条记录，当然可以使用Join和extends来组合使用

```Go
var users []User
err := engine.Where("name = ?", name).And("age > 10").Limit(10, 0).Find(&users)
// SELECT * FROM user WHERE name = ? AND age > 10 limit 0 offset 10

type Detail struct {
    Id int64
    UserId int64 `xorm:"index"`
}

type UserDetail struct {
    User `xorm:"extends"`
    Detail `xorm:"extends"`
}

var users []UserDetail
err := engine.Table("user").Select("user.*, detail.*")
    Join("INNER", "detail", "detail.user_id = user.id").
    Where("user.name = ?", name).Limit(10, 0).
    Find(&users)
// SELECT user.*, detail.* FROM user INNER JOIN detail WHERE user.name = ? limit 0 offset 10
```

* 根据条件遍历数据库，可以有两种方式: Iterate and Rows

```Go
err := engine.Iterate(&User{Name:name}, func(idx int, bean interface{}) error {
    user := bean.(*User)
    return nil
})
// SELECT * FROM user

rows, err := engine.Rows(&User{Name:name})
// SELECT * FROM user
defer rows.Close()
bean := new(Struct)
for rows.Next() {
    err = rows.Scan(bean)
}
```

* ORM方式更新数据，除非使用Cols,AllCols函数指明，默认只更新非空和非0的字段

```Go
affected, err := engine.Id(1).Update(&user)
// UPDATE user SET ... Where id = ?

affected, err := engine.Update(&user, &User{Name:name})
// UPDATE user SET ... Where name = ?

var ids = []int64{1, 2, 3}
affected, err := engine.In(ids).Update(&user)
// UPDATE user SET ... Where id IN (?, ?, ?)

// force update indicated columns by Cols
affected, err := engine.Id(1).Cols("age").Update(&User{Name:name, Age: 12})
// UPDATE user SET age = ?, updated=? Where id = ?

// force NOT update indicated columns by Omit
affected, err := engine.Id(1).Omit("name").Update(&User{Name:name, Age: 12})
// UPDATE user SET age = ?, updated=? Where id = ?

affected, err := engine.Id(1).AllCols().Update(&user)
// UPDATE user SET name=?,age=?,salt=?,passwd=?,updated=? Where id = ?
```

* ORM方式删除记录，需要注意，删除必须至少有一个条件，否则会报错。要清空数据库可以用EmptyTable

```Go
affected, err := engine.Where(...).Delete(&user)
// DELETE FROM user Where ...
```

* ORM方式获取记录条数

```Go
counts, err := engine.Count(&user)
// SELECT count(*) AS total FROM user
```

* <b>Dump数据库结构和数据</b>
DumpAll方法接收一个io.Writer接口来保存Dump出的数据库结构和数据的SQL语句，这个方法导出的SQL语句并不能通用。只针对当前engine所对应的数据库支持的SQL。

```Go
//如果需要在程序中Dump数据库的结构和数据可以使用下面2个方法
engine.DumpAll(w io.Writer)

engine.DumpAllFile(fpath string)
```

* <b>Import 执行数据库SQL脚本</b>
同样，这里需要对应的数据库的SQL语法支持。

```Go
//如果你需要将保存在文件或者其它存储设施中的SQL脚本执行，那么可以使用下面2个方法
engine.Import(r io.Reader)

engine.ImportFile(fpath string)
```

## 部分测试用例


<a href="https://github.com/xormplus/xorm/blob/master/test/xorm_test.go">测试用例</a>，<a href="https://github.com/xormplus/xorm/blob/master/test/测试结果.txt">测试结果</a>

## 文档

* 原版核心ORM功能请详见原版[《xorm操作指南》](http://xorm.io/docs)

## 讨论
请加入QQ群：280360085 进行讨论。API设计相关建议可联系本人QQ：50892683
