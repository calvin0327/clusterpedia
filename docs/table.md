这个提案是想引进对 clusterpedia 的 internal 存储层的分表的设计。clusterpedia 目前使用最多的是 mysql 和 postgresql 数据库存储，之前所有的数据都保存到一张表，随着检索复杂度提升和数据量突增，单表似乎不能满足这样的需求，通过此方案可以灵活的为不同 k8s 的资源来设置不同的数据库表来进行存储，来应对这样的场景。

## 背景：
clusterpedia 内置存储目前是将所有的数据放到一张 resource 表，在小规模的集群规模下和没有复杂查询的情况下，可以满足大部分的需求。但是任然存在一些问题：

* 所有的数据都存放一张表的情况下，对数据检索的性能有一定的损失。例如：k8s event 是一种特殊的数据，他的数量非常大, 会对其他资源的检索有一定性能损失。
* resource 的表结构固定，如果使用非表结构的字段进行检索会虽然可以使用自定义 sql 来满足需求，但是需要用户在上层应用中动态去拼装 sql 语句，增加了使用的复杂性。

## 用户案例

1. 用户场景1
对 event 的 `LastTimestamp` 字段进行排序，这个字段并不在表的字段上。目前我们可以通过 `whereSQL` 使用自定义 sql：

```sql
SELECT JSON_EXTRACT(`object`, '$."lastTimestamp"') FROM `resource` WHERE `group` = '' AND `resource` = 'events' AND `version` = 'v1' AND cluster = 'cluster1' ORDER BY JSON_EXTRACT(`object`, '$."lastTimestamp"')
```

2. 用户场景2

通过名称查询 `Event` 数据，如果存在大量的数据，对资源的名称进行检索，会对数据造成检索压力。

## 实现的目标:

* 第一阶段：通过配置将不同的资源放到不同的数据表
* 第二阶段：对表结构进行扩展，使能够使用普通的 clusterpedia 检索方式，而不需要拼装 sql 语句
* 第三阶段：支持 event 存储到 es 的数据库中

## 实现的细节：

在这个方案中，设计了 3 种分表策略, 分别是 `None`, `GroupResource`, `Custom`. 不同的策略来满足不同的场景：

### 分表的策略

```golang
type Config struct {
	Type    string `env:"DB_TYPE" required:"true"`
	DSN     string `env:"DB_DSN"`
	Port    string `env:"DB_PORT"`
	User    string `env:"DB_USER"`
	...

	DivisionPolicy  DivisionPolicy          `yaml:"divisionPolicy"`
	Mapper          []ResourceMapper       `yaml:"mapper"`
}

type DivisionPolicy string

const (
	None          DivisionPolicy = "None"
	GroupResource DivisionPolicy = "GroupResource"
	Custom        DivisionPolicy = "Custom"
)
```

#### None 分表策略

`None` 是一种默认的配置策略，它是完全兼容之前的存储模式，所有资源都将存到一张表 `resource` 里面去。

#### GroupResource 分表策略

`GroupResource` 是一种可配置的策略。如果配置了此策略，将所有同步的资源自动按照 gr 进行分表。数据库的表名默为 $group_$resource. 数据库表结构将跟 `Resource` 保持一致。

> 在 `GroupResource` 的策略下，将不存在 resource 的表。

#### Custom 分表策略

Custom 是最灵活的也是最复杂的一种表拆分的方式，它不仅支持将不同的资源分拆到不同的表中，也支持扩展 `Resource` 表结构。为了满足这个需求，将增加以下结构来定义表与资源的映射关系：

```golang
type Table struct {
	Name          string `yaml:"name"`
	ExtraFields   []ExtraField `yaml:"extraFields"`
}
```

* Name: 表名，如果是空字符的情况，将使用 $group_$resource 作用表名。
* ExtraFields: 需要扩展表的表结构的字段。

```golang
type ExtraField struct {
	Name      string `yaml:"name"`
	PlainPath string `yaml:"plainPath"`
	Type      string `yaml:"type"`
	Index     string `yaml:"index"`
}
```

* Name：字段名，如果为空字符的话，使用 JSONPath 生成字段名称。
* JSONPath：对应 object 的结构 JSONPath
* Type：数据类型
* Index：是否添加索引
* Tags：类似 gorm 的 tags 信息

```golang
type ResourceMapper struct {
	Table         Table `yaml:"table"`
     SyncResources []clusterv1alpha2.ClusterGroupResources `yaml:"syncResources"`
}
```

* SyncResources: 定义表和 gr 的映射关系。定义的方式跟 `PediaCluster` 类似。

期望的的 yaml 示例：

```yaml
type: mysql
user: root
mapper:
- table:
    name: core
    extraFields:
    - name: uid
      plainPath: /metadata/uid
      type: string
  syncResources:
    resources:
    - pods
    - services
```

> 如果使用 Custom 分表策略，是会存在 resource 表的，如果自定义的 gvr 并没有覆盖同步的资源，剩下的资源将会存储到 resource 表中。

#### 配置持久化

一旦设置了分表的策略后，为了避免表和表结构变动导致数据变动造成的一系列复杂问题，我们需要保证策略在配置后不可变。所以需要将配置信息保存到单独的表 `config` 中。在初始化存储层时，会优先检查数据库中的配置信息是否一致，不一致的情况将会报错。如果一定要修改分表策略，可以通过人为删除表和数据，重新同步。

#### 数据的同步和查询

```golang
type ResourceStorage struct {
	db    *gorm.DB
	codec runtime.Codec

	storageGroupResource schema.GroupResource
	storageVersion       schema.GroupVersion
	memoryVersion        schema.GroupVersion

	table          string
	hasExtralField  bool
}
```

在 `ResourceStorage` 的结构体中增加 `table`, `hasExtralField` 字段。

* table：当前资源对应的表的名称。
* hasExtalField：是否有扩展字段，如果有扩展字段，需要将 object 对 plainPath 进行解析。最后使用 map 的形式保存到数据库中。

无论是同步资源和检索资源，会先定位到具体的表，然后根据 `hasExtralField` 来判断是否需要对数据进行二次处理。
