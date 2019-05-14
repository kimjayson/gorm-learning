## GORM 中文文档

详细文档详情参考 <http://gorm.io/zh_CN/docs/index.html>

使用gorm版本 1.9.1

## 一，快速开始


```

package main
import (
    "log"
    "time"
    "git.xiaojukeji.com/nuwa/gorm"
    _ "github.com/go-sql-driver/mysql"
)
// 默认表名是products， Table 方法可以指定表名
type Product struct {
    Id        int       `gorm:"column:id"`
    Code      string    `gorm:"column:code"` // column 是字段中真实列名， 更过tag ，参见 http://gorm.io/zh_CN/docs/models.html
    Price     int       `gorm:"column:price"`
    CreatedAt time.Time `gorm:"column:created_at"` // 注意时间类型最好使用time.Time 类型，防止格式被转换
    UpdatedAt time.Time `gorm:"column:updated_at"`
}
func main() {
    db, err := gorm.Open("mysql", "root:123456@/test?charset=utf8&parseTime=true&loc=Local")
    if err != nil {
        panic("连接数据库失败")
    }
    defer db.Close()
    // 增 （注意是传的指针！！！如果不传Table, 默认类名 + 's'）
    dbCreate := db.Table("product").Create(&Product{Id: 1, Code: "L1212", Price: 1000, CreatedAt: time.Now(), UpdatedAt: time.Now()})
    // insert into product （`id`, `code`, `price`, `created_at`, `updated_at`） values (1, "L1212", 1000, "", "");
    if dbCreate.Error != nil {
        log.Printf("insert err %v \n", dbCreate.Error)
    }
    // 查
    var product Product
    var products []Product
    dbFind := db.Table("product").Where("id>= ?", 1).Limit(2).Find(&products)
    // select * from product where id >= 1 limit 2;
    if dbFind.Error != nil {
        log.Printf("query err %v \n", dbFind.Error)
    }
    // 改 - 更新product的price为2000
    dbUpdate := db.Table("product").Model(&product).Where("id = ?", 1).Update("Price", 2000)
    // update product set Price = 2000 where id = 1;
    if dbUpdate.Error != nil {
        log.Printf("update err %v \n", dbUpdate.Error)
    }
    // 删
    dbDelete := db.Table("product").Where("id = ?", 2).Delete(&product)
    // delete from product where id = 2;
    if dbDelete.Error != nil {
        log.Printf("delete err %v \n", dbDelete.Error)
    }
    //原生sql, 增
    rowCreate := db.Exec("insert into product (`id`, `code`, `price`, `created_at`, `updated_at`) values (?, ?, ?, ?, ?)", 1, "L1212", 1000, time.Now(), time.Now())
    if rowCreate.Error != nil {
        log.Printf("insert err %v \n", rowCreate.Error)
    }
    //原生sql, 删
    rowDelete := db.Exec("delete from product where id = 2")
    if rowDelete.Error != nil {
        log.Printf("delete err %v \n", rowDelete.Error)
    }
    //原生sql 改
    rowUpdate := db.Exec("update product set Price = ? where id = ?", 2000, 1)
    if rowUpdate.Error != nil {
        log.Printf("update err %v \n", rowUpdate.Error)
    }
    //原生sql 查
    rowSelect := db.Raw("select * from product where id >= ?", 1).Scan(&products)
    if rowSelect.Error != nil {
        log.Printf("select err %v \n", rowSelect.Error)
    }
}
// 注意，gorm 是链式调用，Create, Delete, Update, Find 等方法就会开始发出真实的sql 请求，过滤条件要在这些方法之前！！！
// db.Table("product").Where("id = ?", 1).Delete(&product)
// db.Table("product").Delete(&product).Where("id = ?", 1) 是完全两条不同sql ！！！


```


## 二，使用必读

**1，gorm 是链式调用，注意Create, Delete, Update, Find ，Scan ,Count等方法就会开始发出真实的sql 请求,  返回结果集会保存在对象属性中，过滤条件(where, limit, oder, not, or ,offset等 )，请在这些函数之前！！！Debug 可以打印生成sql 进行验证**

**2，注意Find, Create, Model, Delete 是传的指针！！！**

**3，如果orm不使用Table函数, 默认表名是类名 + 's' ，比如上表就会操作 products 表。**

**4，定义表结构注意时间类型使用time.Time 类型，string 类型会被转换。**

**5，多表连接 ，复杂 sql  对orm 不熟悉 请直接使用原生sql ，推荐使用原生sql。**

**6，如果Orm 给某字段定义了default 默认值，字段为零值将会在插入的时候被忽略！！！。**



## 三，增删改查

### 2.1create 

使用orm时，create 要放到最后


```

// orm insert （注意是传的指针！！！）
dbCreate := db.Table("product").Create(&Product{Id: 1, Code: "L1212", Price: 1000, CreatedAt: time.Now(), UpdatedAt: time.Now()})
// insert into product （`id`, `code`, `price`, `created_at`, `updated_at`） values (1, "L1212", 1000, "", "");
if dbCreate.Error != nil {
    log.Printf("insert err %v \n", dbCreate.Error)
}
 
//原生sql, insert
rowCreate := db.Exec("insert into product (`id`, `code`, `price`, `created_at`, `updated_at`) values (?, ?, ?, ?, ?)", 1, "L1212", 1000, time.Now(), time.Now())
if rowCreate.Error != nil {
    log.Printf("insert err %v \n", rowCreate.Error)
}

```

**注意，如果添加默认值tag,  这种情况下，字段为零值（0， “”， false ） orm insert 将会忽略这个字段，使用默认值，如果有要设置为0的情况，请使用指针类型。**

```

type Animal struct {
    ID   int64
    Name string `gorm:"default:'galeone'"`
    Age  int64
}
 
var animal = Animal{Age: 99, Name: ""}
db.Create(&animal)
// INSERT INTO animals("age") values('99'); // 零值不插入
// SELECT name from animals WHERE ID=111; // 返回主键为 111
// animal.Name => 'galeone'


```

### 2.2delete

使用链式表达，delete 要放到最后

```

// orm 删
dbDelete := db.Table("product").Where("id = ?", 2).Delete(&product)
// delete from product where id = 2;
if dbDelete.Error != nil {
    log.Printf("delete err %v \n", dbDelete.Error)
}
// row sql 删
rowDelete := db.Exec("delete from product where id = 2")
if rowDelete.Error != nil {
    log.Printf("delete err %v \n", rowDelete.Error)
}

```

### 2.3update

使用update， updates ，要放到最后

```

// 改 - 更新product的price为2000
dbUpdate := db.Table("product").Model(&product).Where("id = ?", 1).Update("Price", 2000)
// update product set Price = 2000 where id = 1;
if dbUpdate.Error != nil {
    log.Printf("update err %v \n", dbUpdate.Error)
}
 
//原生sql 改
rowUpdate := db.Exec("update product set Price = ? where id = ?", 2000, 1)
if rowUpdate.Error != nil {
    log.Printf("update err %v \n", rowUpdate.Error)
}

```

**建议不要使用Save， Save 会更新所有字段。**

可以使用Update 更改单个属性，Updates 更新多个属性

```

// Update single attribute if it is changed
db.Model(&user).Update("name", "hello")
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111;
 
// Update single attribute with combined conditions
db.Model(&user).Where("active = ?", true).Update("name", "hello")
//// UPDATE users SET name='hello', updated_at='2013-11-17 21:34:10' WHERE id=111 AND active=true;
 
// Update multiple attributes with `map`, will only update those changed fields
db.Model(&user).Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET name='hello', age=18, actived=false, updated_at='2013-11-17 21:34:10' WHERE id=111;
 
// Update multiple attributes with `struct`, will only update those changed & non blank fields
db.Model(&user).Updates(User{Name: "hello", Age: 18})
//// UPDATE users SET name='hello', age=18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;
 
// WARNING when update with struct, GORM will only update those fields that with non blank value
// For below Update, nothing will be updated as "", 0, false are blank values of their types
db.Model(&user).Updates(User{Name: "", Age: 0, Actived: false})


```

### 2.4 select

**注意过滤条件一定要在Find / First 之前, Find 语句就会将sql 发出！！！**

```

// orm 查
var products []Product
dbFind := db.Table("product").Where("id>= ?", 1).Limit(2).Find(&products)
// select * from product where id >= 1 limit 2;
if dbFind.Error != nil {
    log.Printf("query err %v \n", dbFind.Error)
}
 
//原生sql 查
rowSelect := db.Raw("select * from product where id >= ?", 1).Scan(&products)
if rowSelect.Error != nil {
    log.Printf("select err %v \n", rowSelect.Error)
}

```

where

```

// Get all matched records
db.Where("name = ?", "jinzhu").Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu';
 
// <>
db.Where("name <> ?", "jinzhu").Find(&users)
 
// IN
db.Where("name in (?)", []string{"jinzhu", "jinzhu 2"}).Find(&users)
 
// LIKE
db.Where("name LIKE ?", "%jin%").Find(&users)
 
// AND
db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)
 
// Time
db.Where("updated_at > ?", lastWeek).Find(&users)
 
// BETWEEN
db.Where("created_at BETWEEN ? AND ?", lastWeek, today).Find(&users)

```

**Find 不带Where 条件，极可能产生全表查询，注意！！！**

Struct&Map

```

// Struct
db.Where(&User{Name: "jinzhu", Age: 20}).First(&user)
//// SELECT * FROM users WHERE name = "jinzhu" AND age = 20 LIMIT 1;
 
// Map
db.Where(map[string]interface{}{"name": "jinzhu", "age": 20}).Find(&users)
//// SELECT * FROM users WHERE name = "jinzhu" AND age = 20;
 
// Slice of primary keys
db.Where([]int64{20, 21, 22}).Find(&users)
//// SELECT * FROM users WHERE id IN (20, 21, 22);
 
 
// 零值问题
db.Where(&User{Name: "jinzhu", Age: 0}).Find(&users)
//// SELECT * FROM users WHERE name = "jinzhu";

```

**注意，Where 不推荐使用Struct ，建议使用map，struct 会忽略零值（可以使用指针类型，则不会忽略）**

Not

```

db.Not("name", "jinzhu").First(&user)
//// SELECT * FROM users WHERE name <> "jinzhu" LIMIT 1;
 
// Not In
db.Not("name", []string{"jinzhu", "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name NOT IN ("jinzhu", "jinzhu 2");
 
// Not In slice of primary keys
db.Not([]int64{1,2,3}).First(&user)
//// SELECT * FROM users WHERE id NOT IN (1,2,3);
 
db.Not([]int64{}).First(&user)
//// SELECT * FROM users;
 
// Plain SQL
db.Not("name = ?", "jinzhu").First(&user)
//// SELECT * FROM users WHERE NOT(name = "jinzhu");
 
// Struct
db.Not(User{Name: "jinzhu"}).First(&user)
//// SELECT * FROM users WHERE name <> "jinzhu";

```

Or

```

db.Where("role = ?", "admin").Or("role = ?", "super_admin").Find(&users)
//// SELECT * FROM users WHERE role = 'admin' OR role = 'super_admin';
 
// Struct
db.Where("name = 'jinzhu'").Or(User{Name: "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2';
 
// Map
db.Where("name = 'jinzhu'").Or(map[string]interface{}{"name": "jinzhu 2"}).Find(&users)
//// SELECT * FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2';

```

Select

```

db.Select("name, age").Find(&users)
//// SELECT name, age FROM users;
 
db.Select([]string{"name", "age"}).Find(&users)
//// SELECT name, age FROM users;

```

Order

```
db.Order("age desc, name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;
 
// Multiple orders
db.Order("age desc").Order("name").Find(&users)
//// SELECT * FROM users ORDER BY age desc, name;

```

Limit

```

db.Limit(3).Find(&users)
//// SELECT * FROM users LIMIT 3;

```

Offset

```

db.Offset(3).Find(&users)
//// SELECT * FROM users OFFSET 3;

```

Count

```

db.Where("name = ?", "jinzhu").Or("name = ?", "jinzhu 2").Find(&users).Count(&count)
//// SELECT * from USERS WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (users)
//// SELECT count(*) FROM users WHERE name = 'jinzhu' OR name = 'jinzhu 2'; (count)
 
db.Model(&User{}).Where("name = ?", "jinzhu").Count(&count)
//// SELECT count(*) FROM users WHERE name = 'jinzhu'; (count)
 
db.Table("deleted_users").Count(&count)
//// SELECT count(*) FROM deleted_users;

```

**Count 会调用query，开始真实执行sql 请求，所以第一种情况执行了两条sql，注意放到最后**

Scan

**scan 将结果映射到结构体中去, Scan 会发起真实查询，注意放到语句最后**

```

type Result struct {
    Name string
    Age  int
}
 
var result Result
db.Table("users").Select("name, age").Where("name = ?", 3).Scan(&result)
 
// Raw SQL
db.Raw("SELECT name, age FROM users WHERE name = ?", 3).Scan(&result)

```

##四、压测&&读主

gorm 支持为增删改查添加前缀，可以多次SetPrefix，最终结果会合并成一个map，序列化后当注释插入sql 之前。强制读主，压测，都是通过加前缀功能实现的。/*{"mode":"shadow"}*/  表示 压测， /*{“router”:”m"}*/ 表示读主

**注意SetPrefix 方法必须在Create, First, Find, Exec, Update等执行方法之前。**

```

package main
import (
    "git.xiaojukeji.com/nuwa/gorm"
    _ "github.com/go-sql-driver/mysql"
)
type Product struct {
    gorm.Model
    Code string
    Price uint
}
func main() {
    db, err := gorm.Open("mysql", "user:passwd@/test?charset=utf8&parseTime=true&loc=Local")
    if err != nil {
        panic("连接数据库失败")
    }
    defer db.Close()
    // 创建
    db.SetPrefix("mode", "shadow").Create(&Product{Code: "L1212", Price: 1000})
    // /*{"mode":"shadow"}*/  insert into products (`code`, `price`) valus ("L1212", 1000)
    // 读取
    var product Product
    db.SetPrefix("router", "m").First(&product, 1) // 查询id为1的product
    // /*{“router”:”m"}*/ select * from products where `id` =1
    db.SetPrefix("router", "m").First(&product, "code = ?", "L1212") // 查询code为l1212的product
    // /*{“router”:”m"}*/ select * from products where `code` = "L1212"
    // 删除 - 删除product
    db.SetPrefix("router", "m").Delete(&product)
    // /*{“router”:”m"}*/ delete from products;
    // 原生sql
    db.SetPrefix("router", "m").Exec("SELECT * FROM users;")
    // /*{“router”:”m"}*/"SELECT * FROM users;"
}

```

## 五、原生sql

相比orm ，推荐使用原生sql 

```

5.1. 执行原生SQL
db.Exec("DROP TABLE users;")
db.Exec("UPDATE orders SET shipped_at=? WHERE id IN (?)", time.Now, []int64{11,22,33})
// Scan
type Result struct {
    Name string
    Age int
}
var result Result
db.Raw("SELECT name, age FROM users WHERE name = ?", 3).Scan(&result)
 
5.2. sql.Row & sql.Rows
获取查询结果为*sql.Row或*sql.Rows
row := db.Table("users").Where("name = ?", "jinzhu").Select("name, age").Row() // (*sql.Row)
row.Scan(&name, &age)
rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Select("name, age, email").Rows() // (*sql.Rows, error)
defer rows.Close()
for rows.Next() {
    ...
    rows.Scan(&name, &age, &email)
    ...
}
// Raw SQL
rows, err := db.Raw("select name, age, email from users where name = ?", "jinzhu").Rows() // (*sql.Rows, error)
defer rows.Close()
for rows.Next() {
    ...
    rows.Scan(&name, &age, &email)
    ...
}
 
5.3. 迭代中使用sql.Rows的Scan
rows, err := db.Model(&User{}).Where("name = ?", "jinzhu").Select("name, age, email").Rows() // (*sql.Rows, error)
defer rows.Close()
for rows.Next() {
    var user User
    db.ScanRows(rows, &user)
    // do something
}

```

## 六、事务

参考grom文档

```

6.1. 执行事务

// 开始事务
tx := db.Begin()

// 在事务中做一些数据库操作（从这一点使用'tx'，而不是'db'）
tx.Create(...)
tx.Find(...)

// do something...

// 发生错误时回滚事务
tx.Rollback()

// 或提交事务
tx.Commit()

```
