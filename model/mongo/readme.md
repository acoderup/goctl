# mongo生成model

## 背景

在业务务开发中，model(dao)数据访问层是一个服务必不可缺的一层，因此数据库访问的CURD也是必须要对外提供的访问方法，
而CURD在go-zero中就仅存在两种情况

* 带缓存model
* 不带缓存model

从代码结构上来看，C-U-R-D四个方法就是固定的结构，因此我们可以将其交给goctl工具去完成，帮助我们提升开发效率。

## 方案设计

mongo的生成不同于mysql，mysql可以从scheme_information库中读取到一张表的信息（字段名称，数据类型，索引等），
而mongo是文档型数据库，我们暂时无法从db中读取某一条记录来实现字段信息获取，就算有也不一定是完整信息（某些字段可能是omitempty修饰，可有可无），
这里采用type自己编写+代码生成方式实现

## 使用示例

为 User 生成 mongo model

```bash
$ goctl model mongo -t User -c --dir .
```

### 生成示例代码

#### usermodel.go

```go
package model

import (
	"github.com/zeromicro/go-zero/core/stores/cache"
	"github.com/zeromicro/go-zero/core/stores/monc"
)

var _ UserModel = (*customUserModel)(nil)

type (
	// UserModel is an interface to be customized, add more methods here,
	// and implement the added methods in customUserModel.
	UserModel interface {
		userModel
	}

	customUserModel struct {
		*defaultUserModel
	}
)

// NewUserModel returns a model for the mongo.
func NewUserModel(url, db, collection string, c cache.CacheConf) UserModel {
	conn := monc.MustNewModel(url, db, collection, c)
	return &customUserModel{
		defaultUserModel: newDefaultUserModel(conn),
	}
}

```

#### usermodelgen.go

```go
// Code generated by goctl. DO NOT EDIT.
package model

import (
	"context"
	"time"

	"github.com/zeromicro/go-zero/core/stores/monc"
	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/bson/primitive"
	"go.mongodb.org/mongo-driver/mongo"
)

var prefixUserCacheKey = "cache:user:"

type userModel interface {
	Insert(ctx context.Context, data *User) error
	FindOne(ctx context.Context, id string) (*User, error)
	Update(ctx context.Context, data *User) (*mongo.UpdateResult, error)
	Delete(ctx context.Context, id string) (int64, error)
}

type defaultUserModel struct {
	conn *monc.Model
}

func newDefaultUserModel(conn *monc.Model) *defaultUserModel {
	return &defaultUserModel{conn: conn}
}

func (m *defaultUserModel) Insert(ctx context.Context, data *User) error {
	if !data.ID.IsZero() {
		data.ID = primitive.NewObjectID()
		data.CreateAt = time.Now()
		data.UpdateAt = time.Now()
	}

	key := prefixUserCacheKey + data.ID.Hex()
	_, err := m.conn.InsertOne(ctx, key, data)
	return err
}

func (m *defaultUserModel) FindOne(ctx context.Context, id string) (*User, error) {
	oid, err := primitive.ObjectIDFromHex(id)
	if err != nil {
		return nil, ErrInvalidObjectId
	}

	var data User
	key := prefixUserCacheKey + id
	err = m.conn.FindOne(ctx, key, &data, bson.M{"_id": oid})
	switch err {
	case nil:
		return &data, nil
	case monc.ErrNotFound:
		return nil, ErrNotFound
	default:
		return nil, err
	}
}

func (m *defaultUserModel) Update(ctx context.Context, data *User) (*mongo.UpdateResult, error) {
	data.UpdateAt = time.Now()
	key := prefixUserCacheKey + data.ID.Hex()
	res, err := m.conn.ReplaceOne(ctx, key, bson.M{"_id": data.ID}, bson.M{"$set": data})
	return res, err
}

func (m *defaultUserModel) Delete(ctx context.Context, id string) (int64, error) {
	oid, err := primitive.ObjectIDFromHex(id)
	if err != nil {
		return 0, ErrInvalidObjectId
	}
	key := prefixUserCacheKey + id
	res, err := m.conn.DeleteOne(ctx, key, bson.M{"_id": oid})
	return res, err
}

```

#### usertypes.go

```go
package model

import (
	"time"

	"go.mongodb.org/mongo-driver/bson/primitive"
)

type User struct {
	ID primitive.ObjectID `bson:"_id,omitempty" json:"id,omitempty"`
	// TODO: Fill your own fields
	UpdateAt time.Time `bson:"updateAt,omitempty" json:"updateAt,omitempty"`
	CreateAt time.Time `bson:"createAt,omitempty" json:"createAt,omitempty"`
}

```

#### error.go

```go
package model

import (
	"errors"

	"github.com/zeromicro/go-zero/core/stores/mon"
)

var (
	ErrNotFound        = mon.ErrNotFound
	ErrInvalidObjectId = errors.New("invalid objectId")
)
```

### 文件目录预览

```text
.
├── error.go
├── usermodel.go
├── usermodelgen.go
└── usertypes.go
```

## 命令预览

```text
Generate mongo model

Usage:
  goctl model mongo [flags]

Flags:
      --branch string   The branch of the remote repo, it does work with --remote
  -c, --cache           Generate code with cache [optional]
  -d, --dir string      The target dir
  -h, --help            help for mongo
      --home string     The goctl home path of the template, --home and --remote cannot be set at the same time, if they are, --remote has higher priority
      --remote string   The remote git repo of the template, --home and --remote cannot be set at the same time, if they are, --remote has higher priority
                                The git repo directory must be consistent with the https://github.com/zeromicro/go-zero-template directory structure
      --style string    The file naming format, see [https://github.com/zeromicro/go-zero/tree/master/tools/goctl/config/readme.md]
  -t, --type strings    Specified model type name

```

> 温馨提示
>
> `--type` 支持slice传值，示例 `goctl model mongo -t=User -t=Class`

