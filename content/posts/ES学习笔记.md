# ES学习笔记

## ES基本概念

* 使⽤ Java 编写的⼀种开源搜索引擎，它在内部使⽤ Luence 做索引与搜索，Lucene是一套用于全文检索和搜寻的开源程序库，由Apache软件基金会支持和提供。ES通过对 Lucene 的封装，提供了⼀套简单⼀致的 RESTful API。其中全文检索就是利用倒排索引技术对需要搜索的数据进行处理，然后提供快速的全文匹配的技术。
* 是⼀个可⾼度伸缩的开源数据存储、全⽂搜索和数据分析引擎，ES常用于全文搜索与高亮显示、日志收集与分析、个性化推荐系统、商品价格区间统计与筛选以及地理位置搜索等。
* ⼀种分布式的搜索引擎架构，可以很简单地扩展到上百个服务节点，并⽀持 PB 级别的数据查询，使系统具备⾼可⽤和⾼并发性。

## 倒排索引

* 假设有这样的几个文档，我们基于`id`和`title`构建倒排索引：

  | id   | title                                      |
  | ---- | ------------------------------------------ |
  | 1    | 谷歌地图之父跳槽FaceBook                   |
  | 2    | 谷歌地图之父加盟FaceBook                   |
  | 3    | 谷歌地图创始人拉斯离开谷歌加盟Facebook     |
  | 4    | 谷歌地图之父跳槽Facebook与Wave项目取消有关 |
  | 5    | 谷歌地图之父拉斯加盟社交网站Facebook       |

  ---

  1. 对 `title` 进行分词。
  2. 为每个词建立映射到文档 ID 的关系。

  ---

  假设以每个词为单位分词：
  - **文档 1**: `谷歌`、`地图`、`之父`、`跳槽`、`FaceBook`
  - **文档 2**: `谷歌`、`地图`、`之父`、`加盟`、`FaceBook`
  - **文档 3**: `谷歌`、`地图`、`创始人`、`拉斯`、`离开`、`谷歌`、`加盟`、`Facebook`
  - **文档 4**: `谷歌`、`地图`、`之父`、`跳槽`、`Facebook`、`与`、`Wave`、`项目`、`取消`、`有关`
  - **文档 5**: `谷歌`、`地图`、`之父`、`拉斯`、`加盟`、`社交`、`网站`、`Facebook`

  ---

  构建倒排索引表：

  | 词       | 文档 ID       |
  | -------- | ------------- |
  | 谷歌     | 1, 2, 3, 4, 5 |
  | 地图     | 1, 2, 3, 4, 5 |
  | 之父     | 1, 2, 4, 5    |
  | 跳槽     | 1, 4          |
  | 加盟     | 2, 3, 5       |
  | FaceBook | 1, 2          |
  | Facebook | 3, 4, 5       |
  | 创始人   | 3             |
  | 拉斯     | 3, 5          |
  | 离开     | 3             |
  | 与       | 4             |
  | Wave     | 4             |
  | 项目     | 4             |
  | 取消     | 4             |
  | 有关     | 4             |
  | 社交     | 5             |
  | 网站     | 5             |

  ---

  搜索过程：

  1. 当用户输入任意的内容时，首先对用户输入的内容进行分词，得到用户要搜索的所有词条。
  2. 然后拿着这些词条去倒排索引表中进行匹配，找到这些词条就能找到包含这些词条的所有文档的编号。
  3. 然后根据这些编号去文档列表中找到文档。

  比如搜索：拉斯跳槽

  1. 首先对这句话进行分词，得到两个词条：拉斯、跳槽。
  2. 然后去倒排索引表搜索，得到这两个词所在的文档编号1, 3, 4, 5。
  3. 然后根据编号到文档列表查找，就可以得到原始文档的信息了。

  | id   | title                                      |
  | ---- | ------------------------------------------ |
  | 1    | 谷歌地图之父跳槽FaceBook                   |
  | 3    | 谷歌地图创始人拉斯离开谷歌加盟Facebook     |
  | 4    | 谷歌地图之父跳槽Facebook与Wave项目取消有关 |
  | 5    | 谷歌地图之父拉斯加盟社交网站Facebook       |

## 核心概念

* **Cluster**：集群，由一个或多个`ElasticSearch`节点组成。
  * 节点只能通过集群名称进行加入。
  * 不同的环境中使用不同的集群名称。
* **Node**：节点，组成`ElasticSearch`集群的服务单元，同一个集群内节点的名字不能重复，通常在一个节点上分配一个或多个分片。
  * 是一个`ES`的运行实例，即进程。
  * 节点存储数据，并参与集群的索引、搜索和分析功能。
  * 启动时分配给节点的随机通用唯一标识符UUID为默认节点名称，也可以指定有意义的节点索引名称。
* **Shards**：分片，当索引上的数据量太大的时候，我们通常会将⼀个索引上的数据进⾏⽔平拆分，拆分出来的每个数据库叫作⼀个分⽚。
  * 在一个多分片的索引中写入数据时，通过路由来确定具体写入哪一个分片中，所以在创建索引时需要指定分片的数量，并且分片的数量一旦确定就不能再修改。
  * 分片后的索引带来了规模上（数据水平切分）的和性能上（并行执行）的提升。
  * 每个分⽚都是` Luence` 中的⼀个索引⽂件，每个分⽚必须有⼀个主分⽚和零到多个副本分⽚。
* **Replicas**：备份也叫作副本，是指对主分片的备份，主分片和备份分片都可以对外提供查询服务，写操作时先在主分片上完成，然后发送到备份上。
  * 当主分⽚不可⽤时，会在备份的分⽚中选举出⼀个作为主分⽚，所以备份不仅可以提升系统的⾼可⽤性能，还可以提升搜索时的并发性能。
  * 若副本太多的话，在写操作时会增加数据同步的负担；副本太多，也会造成资源紧张。
* **Index**：索引，由一个或多个分片组成，通过索引的名字在集群内进行唯一标识。
  * 具有某种相似特征的文档集合。
  * 当对其中的文档执行索引、搜索、查询和删除操作时，通过名称标识指向这个特定的索引。
  * 名称标识为小写。
* **Document**：文档，索引中的每一条数据叫作一个文档，类似于关系数据库中的一条数据，通过`_id`在`Type`内进行唯一标识。
  * 可以被索引的基本信息单元。
  * ⽂档以JSON表示。
  * 在单个索引中，理论上可以存储任意多的⽂档。

## 核心功能

* 索引与文档管理
  * 索引的创建与管理（创建、更新、删除） 
  * 文档操作（添加文档、查询文档、更新文档和删除文档）
* 搜索功能
  * 简单查询（查询某个字段等于某个值的记录）、精准匹配（用于匹配特定的关键词）、布尔查询（组合多条件查询）
* 聚合功能
  * 指标聚合（统计某个字段的最大值、最小值、平均值、总和等）
  * 分桶聚合（按字段值分组）
  * 嵌套聚合（结合分桶聚合和指标聚合）
* 自动补全

## 安装部署

* Docker部署：

docker network create es-net

1. `docker network create es-net`：创建docker网络，这里就是定义一个默认的桥接网络叫es-net，用于让容器在相同的网络中互通

docker run -d --name es     -e "ES_JAVA_OPTS=-Xms512m -Xmx512m"     -e "discovery.type=single-node"     -v es-data:/usr/share/elasticsearch/data     -v es-plugins:/usr/share/elasticsearch/plugins     --privileged     --network es-net     -p 9200:9200     -p 9300:9300 elasticsearch:7.12.1

2. `docker run -d --name es ... elasticsearch:7.12.1`：运行Docker容器，-d是以后台模式运行容器，--name es是给容器命名es，elasticsearch:7.12.1是制定使用的镜像和版本
3. `-e "ES_JAVA_OPTS=-Xms512m -Xmx512m"     -e "discovery.type=single-node"`：环境变量设置
4. `-v es-data:/usr/share/elasticsearch/data     -v es-plugins:/usr/share/elasticsearch/plugins`：数据卷挂载，就是将容器内的目录与宿主机上面的路径绑定，用于数据持久化
5. `--network es-net`：设置Docker网络，指定容器使用刚才创建的`es-net`网络
6. `-p 9200:9200 -p 9300:9300`：端口映射，将宿主机的端口与容器内的端口绑定，那么宿主机就可以通过`localhost:9200`来访问es了

* 主要目录：

  ![image-20241119154330112](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241119154330112.png)

  * `bin`：存放可执行文件，例如启动脚本、密钥工具等。
  * `config`：ES所有的配置文件都在这个目录下。
  * `data`：默认的索引数据存储位置。
  * `log`：默认的日志。

## Elasticsearch 常用操作以及对应Golang代码(olivere/elastic/v7库实现)

### 获取ES连接

```go
var (
	esOnce sync.Once
	esCli  *elastic.Client
)

func GetESClient() *elastic.Client {
	if esCli != nil {
		return esCli
	}
	esOnce.Do(func() {
		cli, err := elastic.NewSimpleClient(
			elastic.SetURL("http://XXX.XXX.XXX.XXX:9200"),
			elastic.SetErrorLog(log.New(os.Stderr, "", log.LstdFlags)),
			elastic.SetInfoLog(log.New(os.Stdout, "", log.LstdFlags)))
		if err != nil {
			log.Fatalln("failed to new simple client: ", err)
		}
		esCli = cli
	})
	return esCli
}
```

## 1. 索引库操作

### 1.1 创建索引库和映射

#### DSL语法

```json
PUT /consumer
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "sex": {
        "type": "keyword"
      },
      "hobby": {
        "properties": {
          "sports": {
            "type": "keyword"
          },
          "music": {
            "type": "keyword"
          },
          "reading": {
            "type": "keyword"
          }
        }
      },
      "age": {
        "type": "integer"
      },
      "email": {
        "type": "keyword"
      },
      "address": {
        "type": "text"
      }
    }
  }
}
```

#### 对应的Golang操作

```go
// CrtESIndex 创建索引
func CrtESIndex(ctx context.Context, index, desc string) error {
	exist, err := ESIndexExist(ctx, index)
	if err != nil {
		return err
	}
	if exist {
		return nil
	}
	_, err = client.GetESClient().CreateIndex(index).BodyString(desc).Do(ctx)
	return err
}

// ESIndexExist 判断索引是否存在
func ESIndexExist(ctx context.Context, index string) (bool, error) {
	return client.GetESClient().IndexExists(index).Do(ctx)
}
```

#### Golang测试代码

```go
func TestCreateIndex(t *testing.T) {
	desc := `
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "sex": {
        "type": "keyword"
      },
      "hobby": {
        "properties": {
          "sports": {
            "type": "keyword"
          },
          "music": {
            "type": "keyword"
          },
          "reading": {
            "type": "keyword"
          }
        }
      },
      "age": {
        "type": "integer"
      },
      "email": {
        "type": "keyword"
      },
      "address": {
        "type": "text"
      }
    }
  }
}`
	err := CrtESIndex(context.Background(), "consumer", desc)
	require.NoError(t, err)
}
```



### 1.2 查询索引库

#### DSL语法

```json
GET /consumer
```

#### 对应的Golang操作

```go
// GetESIndex 获取索引信息
func GetESIndex(ctx context.Context, index string) (string, error) {
	res, err := client.GetESClient().IndexGet().Index(index).Do(ctx)
	if err != nil {
		return "", err
	}
	jsonBytes, err := json.MarshalIndent(res, "", "  ")
	if err != nil {
		return "", err
	}
	return string(jsonBytes), nil
}
```

#### Golang测试代码

```go
func TestGetESIndex(t *testing.T) {
	// 创建consumer1索引
	TestCreateIndex(t)
	index := "consumer"
	res, err := GetESIndex(context.Background(), index)
	require.NoError(t, err)
	log.Println(res)
}
```

### 1.3 修改索引库

#### DSL语法

> 注意：索引库一旦创建，就无法修改已有的mapping，但是允许向mapping中添加新的字段

```json
PUT /consumer/_mapping
{
  "properties":{
    "phone":{
      "type": "keyword"
    }
  }
}
```

### 1.4 删除索引库

#### DSL语法

```json
DELETE /consumer
```

#### 对应的Golang操作

```go
// DelESIndex 删除索引
func DelESIndex(ctx context.Context, index string) error {
	_, err := client.GetESClient().DeleteIndex(index).Do(ctx)
	return err
}

```

#### Golang测试代码

```go
func TestDelESIndex(t *testing.T) {
	index := "consumer"
	TestCreateIndex(t)
	err := DelESIndex(context.Background(), index)
	require.NoError(t, err)
}
```

## 2. 文档操作

### 2.1 新增文档

#### DSL语法

```json
POST /consumer/_doc/1
{
  "name": "张三",
  "sex": "男",
  "age": 28,
  "email": "zhangsan@example.com",
  "phone": "13800138000",
  "address": "北京市朝阳区",
  "hobby": {
    "music": "流行",
    "reading": "小说",
    "sports": "篮球"
  }
}
```

#### 对应的Golang操作

```go
// AddDoc 新增文档
func AddDoc(ctx context.Context, index, id string, data interface{}) (err error) {
	_, err = client.GetESClient().Index().
		Index(index).
		Id(id).
		BodyJson(data).
		Do(ctx)
	return
}
```

#### Golang测试代码

```go
func TestAddDoc(t *testing.T) {
	// _ = index.CrtESIndex(context.Background(), "consumer", desc)
	consumer := map[string]interface{}{
		"name":    "张三",
		"sex":     "男",
		"age":     28,
		"email":   "zhangsan@example.com",
		"phone":   "13800138000",
		"address": "北京市朝阳区",
		"hobby": map[string]interface{}{
			"music":   "流行",
			"reading": "小说",
			"sports":  "篮球",
		},
	}
	err := AddDoc(context.Background(), "consumer", "1", consumer)
	require.NoError(t, err)
}
```

### 2.2 查询文档

#### DSL语法

```json
// 查询单个文档
GET /consumer/_doc/1

// 查询索引库下的所有文档
GET /consumer/_search
```

#### 对应的Golang操作

```go
// GetDoc 获取文档
func GetDoc(ctx context.Context, index, id string) (data map[string]interface{}, err error) {
	resp, err := client.GetESClient().Get().
		Index(index).
		Id(id).
		Do(ctx)
	if err != nil {
		return nil, err
	}

	// 如果文档不存在
	if !resp.Found {
		return nil, nil
	}

	// 将文档源数据解析为 map
	err = json.Unmarshal(resp.Source, &data)
	return data, err
}
```

#### Golang测试代码

```go
func TestGetDoc(t *testing.T) {
	data, err := GetDoc(context.Background(), "consumer", "1")
	require.NoError(t, err)
	t.Log(data)
}
```

### 2.3 删除文档

#### DSL语法

```json
DELETE /consumer/_doc/1
```

#### 对应的Golang操作

```go
// DelDoc 删除文档
func DelDoc(ctx context.Context, index, id string) error {
	_, err := client.GetESClient().Delete().
		Index(index).
		Id(id).
		Do(ctx)
	return err
}
```

#### Golang测试代码

```go
func TestDelDoc(t *testing.T) {
	TestAddDoc(t)
	err := DelDoc(context.Background(), "consumer", "1")
	require.NoError(t, err)
	exist, err := ExistDoc(context.Background(), "consumer", "1")
	require.NoError(t, err)
	require.Equal(t, false, exist)
}
```

### 2.4 修改文档

#### 全量修改
```json
PUT /consumer/_doc/1
{
  "name": "张三",
  "sex": "男",
  "age": 30,
  "email": "zhangsan@example.com",
  "phone": "13800138000",
  "address": "北京市朝阳区",
  "hobby": {
    "music": "流行",
    "reading": "小说",
    "sports": "篮球"
  }
}
```

#### 增量修改

```json
POST /consumer/_update/1
{
  "doc":{
    "age": 32
  }
}
```

#### 对应的Golang操作

```go
func UpdateDoc(ctx context.Context, index, id string, data interface{}) error {
	_, err := client.GetESClient().Update().
		Index(index).
		Id(id).
		Doc(data).
		Do(ctx)
	return err
}
```

#### Golang测试代码

```go
func TestUpdateDoc(t *testing.T) {
	TestAddDoc(t)
	consumer := map[string]interface{}{
		"name": "李四",
	}
	err := UpdateDoc(context.Background(), "consumer", "1", consumer)
	require.NoError(t, err)
	data, err := GetDoc(context.Background(), "consumer", "1")
	require.NoError(t, err)
	t.Log(data)
}
```

## 3. 查询操作

### 3.1 全文检索查询

#### match_all 查询所有
```json
GET /consumer/_search
{
  "query": {
    "match_all": {}
  }
}
```

#### 对应的Golang操作

```go
func MatchAllQuery(ctx context.Context, index string) (data map[string]interface{}, err error) {
	// 创建 match_all 查询
	query := elastic.NewMatchAllQuery()

	// 执行搜索
	res, err := client.GetESClient().Search().
		Index(index).
		Query(query).
		Size(10000). // 设置返回的最大文档数，根据需求调整
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 初始化返回结果
	data = make(map[string]interface{})

	// 添加基本信息到结果中
	data["total"] = res.TotalHits()
	data["took"] = res.TookInMillis

	// 处理搜索结果
	var hits []map[string]interface{}
	for _, hit := range res.Hits.Hits {
		// 将文档源数据解析为 map
		var docData map[string]interface{}
		err = json.Unmarshal(hit.Source, &docData)
		if err != nil {
			return nil, err
		}

		// 添加文档元数据
		docData["_id"] = hit.Id
		docData["_score"] = hit.Score

		hits = append(hits, docData)
	}

	// 将处理后的hits添加到结果中
	data["hits"] = hits

	return data, nil
}
```

#### Golang测试代码

```go
func TestMatchAllQuery(t *testing.T) {
	data, err := MatchAllQuery(context.Background(), "consumer")
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

#### match 单字段查询

```json
GET /consumer/_search
{
  "query": {
    "match": {
      "name": "张"
    }
  }
}
```

#### 对应的Golang操作

```go
func MatchQuery(ctx context.Context, index, field string, value interface{}) (data map[string]interface{}, err error) {
	// 创建 match 查询
	query := elastic.NewMatchQuery(field, value)

	// 执行搜索
	res, err := client.GetESClient().Search().
		Index(index).
		Query(query).
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 初始化返回结果
	data = make(map[string]interface{})

	// 添加基本信息到结果中
	data["total"] = res.TotalHits()
	data["took"] = res.TookInMillis

	// 处理搜索结果
	var hits []map[string]interface{}
	for _, hit := range res.Hits.Hits {
		// 将文档源数据解析为 map
		var docData map[string]interface{}
		err = json.Unmarshal(hit.Source, &docData)
		if err != nil {
			return nil, err
		}

		// 添加文档元数据
		docData["_id"] = hit.Id
		docData["_score"] = hit.Score

		hits = append(hits, docData)
	}

	// 将处理后的hits添加到结果中
	data["hits"] = hits

	return data, nil
}
```

#### Golang测试代码

```go
func TestMatchQuery(t *testing.T) {
	data, err := MatchQuery(context.Background(), "consumer", "name", "三")
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

#### multi_match 多字段查询

```json
GET /consumer/_search
{
  "query": {
    "multi_match": {
      "query": "张",
      "fields": ["name", "sex", "address"]
    }
  }
}
```

#### 对应的Golang操作

```go
func MultiMatchQuery(ctx context.Context, index string, queryText string, fields []string) (data map[string]interface{}, err error) {
	// 创建 multi_match 查询
	query := elastic.NewMultiMatchQuery(queryText, fields...).
		Type("best_fields") // 可以根据需要修改类型：best_fields, most_fields, cross_fields 等

	// 执行搜索
	res, err := client.GetESClient().Search().
		Index(index).
		Query(query).
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 初始化返回结果
	data = make(map[string]interface{})

	// 添加基本信息到结果中
	data["total"] = res.TotalHits()
	data["took"] = res.TookInMillis

	// 处理搜索结果
	var hits []map[string]interface{}
	for _, hit := range res.Hits.Hits {
		// 将文档源数据解析为 map
		var docData map[string]interface{}
		err = json.Unmarshal(hit.Source, &docData)
		if err != nil {
			return nil, err
		}

		// 添加文档元数据
		docData["_id"] = hit.Id
		docData["_score"] = hit.Score

		hits = append(hits, docData)
	}

	// 将处理后的hits添加到结果中
	data["hits"] = hits

	return data, nil
}
```

#### Golang测试代码

```go
func TestMultiMatchQuery(t *testing.T) {
	data, err := MultiMatchQuery(context.Background(), "consumer", "张", []string{"name", "address"})
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

### 3.2 精准查询

#### term 词条精准查询
```json
GET /consumer/_search
{
  "query": {
    "term": {
      "email": {
        "value": "zhangsan@example.com"
      }
    }
  }
}
```

#### 对应的Golang操作

```go
func TermQuery(ctx context.Context, index string, field string, value interface{}) (data map[string]interface{}, err error) {
	// 创建 term 查询
	query := elastic.NewTermQuery(field, value)

	// 执行搜索
	res, err := client.GetESClient().Search().
		Index(index).
		Query(query).
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 初始化返回结果
	data = make(map[string]interface{})

	// 添加基本信息到结果中
	data["total"] = res.TotalHits()
	data["took"] = res.TookInMillis

	// 处理搜索结果
	var hits []map[string]interface{}
	for _, hit := range res.Hits.Hits {
		// 将文档源数据解析为 map
		var docData map[string]interface{}
		err = json.Unmarshal(hit.Source, &docData)
		if err != nil {
			return nil, err
		}

		// 添加文档元数据
		docData["_id"] = hit.Id
		docData["_score"] = hit.Score

		hits = append(hits, docData)
	}

	// 将处理后的hits添加到结果中
	data["hits"] = hits

	return data, nil
}
```

#### Golang测试代码

```go
func TestTermQuery(t *testing.T) {
	data, err := TermQuery(context.Background(), "consumer", "email", "zhangsan@example.com")
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

#### range 范围查询

```json
GET /consumer/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 32,
        "lte": 34
      }
    }
  }
}
```

#### 对应的Golang操作

```go
func RangeQuery(ctx context.Context, index string, field string, gte, lte interface{}) (data map[string]interface{}, err error) {
	// 创建 range 查询
	query := elastic.NewRangeQuery(field).
		Gte(gte).
		Lte(lte)

	// 执行搜索
	res, err := client.GetESClient().Search().
		Index(index).
		Query(query).
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 初始化返回结果
	data = make(map[string]interface{})

	// 添加基本信息到结果中
	data["total"] = res.TotalHits()
	data["took"] = res.TookInMillis

	// 处理搜索结果
	var hits []map[string]interface{}
	for _, hit := range res.Hits.Hits {
		// 将文档源数据解析为 map
		var docData map[string]interface{}
		err = json.Unmarshal(hit.Source, &docData)
		if err != nil {
			return nil, err
		}

		// 添加文档元数据
		docData["_id"] = hit.Id
		docData["_score"] = hit.Score

		hits = append(hits, docData)
	}

	// 将处理后的hits添加到结果中
	data["hits"] = hits

	return data, nil
}
```

#### Golang测试代码

```go
func TestRangeQuery(t *testing.T) {
	data, err := RangeQuery(context.Background(), "consumer", "age", 20, 30)
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

### 3.4 复合查询

复合查询主要包含以下几种类型：
- `function_score`：允许修改查询结果中每个文档的分数
- `bool`：组合多个查询条件，包含：
  - `must`：必须匹配
  - `should`：可选匹配，匹配则增加得分
  - `must_not`：必须不匹配
  - `filter`：过滤文档，不影响得分

#### 复合查询示例
```json
GET /hotel/_search
{
  "query": {
    "function_score": {
      "query": {
        "bool": {
          "must": [
            {"term": {"city": "北京"}}
          ],
          "should": [
            {"term": {"brand": "希尔顿"}},
            {"term": {"brand": "万豪"}}
          ],
          "must_not": [
            {"range": {"price": {"lte": 500}}}
          ],
          "filter": [
            {"range": {"score": {"gte": 45}}}
          ]
        }
      },
      "functions": [
        {
          "filter": {
            "term": {
              "brand": "汉庭"
            }
          },
          "weight": 2
        }
      ],
      "boost_mode": "sum"
    }
  }
}
```

#### 对应的Golang操作

```go
func HotelSearch(ctx context.Context) (data map[string]interface{}, err error) {
	// 1. 创建基础的布尔查询
	boolQuery := elastic.NewBoolQuery().
		Must(elastic.NewTermQuery("city", "北京")).
		Should(
			elastic.NewTermQuery("brand", "希尔顿"),
			elastic.NewTermQuery("brand", "万豪"),
		).
		MustNot(elastic.NewRangeQuery("price").Lte(500)).
		Filter(elastic.NewRangeQuery("score").Gte(45))

	// 2. 创建 function score 查询
	functionScoreQuery := elastic.NewFunctionScoreQuery().
		Query(boolQuery).
		Add(elastic.NewTermQuery("brand", "汉庭"), elastic.NewWeightFactorFunction(2)).
		BoostMode("sum")

	// 3. 执行搜索
	result, err := client.GetESClient().Search().
		Index("hotel").
		Query(functionScoreQuery).
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 4. 处理结果
	data = make(map[string]interface{})

	data["total"] = result.TotalHits()
	data["took"] = result.TookInMillis

	var hits []map[string]interface{}
	for _, hit := range result.Hits.Hits {
		var docData map[string]interface{}
		if err := json.Unmarshal(hit.Source, &docData); err != nil {
			return nil, err
		}

		docData["_id"] = hit.Id
		docData["_score"] = hit.Score
		hits = append(hits, docData)
	}

	data["hits"] = hits

	return data, nil
}
```

#### Golang测试代码

```go
func TestHotelSearch(t *testing.T) {
	data, err := HotelSearch(context.Background())
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

## 4. 搜索结果处理

搜索DSL包含以下主要属性：
- query：查询条件
- from和size：分页条件
- sort：排序条件
- highlight：高亮条件
- aggs：聚合条件

### 4.1 排序
```json
GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "price": {
        "order": "desc"
      },
      "score": {
        "order": "asc"
      }
    }
  ]
}
```

#### 对应的Golang操作

```go
func HotelSearchSort(ctx context.Context) (data map[string]interface{}, err error) {
	// 1. 创建查询
	matchAllQuery := elastic.NewMatchAllQuery()

	// 2. 设置排序条件
	sortByPrice := elastic.NewFieldSort("price").Order(false) // false 表示降序
	sortByScore := elastic.NewFieldSort("score").Order(true)  // true 表示升序

	// 3. 执行搜索
	result, err := client.GetESClient().Search().
		Index("hotel").
		Query(matchAllQuery).
		SortBy(sortByPrice, sortByScore). // 添加排序条件
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 4. 处理结果
	data = make(map[string]interface{})
	data["total"] = result.TotalHits()
	data["took"] = result.TookInMillis

	var hits []map[string]interface{}
	for _, hit := range result.Hits.Hits {
		var docData map[string]interface{}
		if err := json.Unmarshal(hit.Source, &docData); err != nil {
			continue
		}

		docData["_id"] = hit.Id
		docData["_score"] = hit.Score
		hits = append(hits, docData)
	}

	data["hits"] = hits

	return data, nil
}
```

#### Golang测试代码

```go
func TestHotelSearchSort(t *testing.T) {
	data, err := HotelSearchSort(context.Background())
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

### 4.2 分页

```json
GET /hotel/_search
{
  "query": {
    "match_all": {}
  },
  "from": 0,
  "size": 2
}
```

#### 对应的Golang操作

```go
func HotelSearchPaging(ctx context.Context) (data map[string]interface{}, err error) {
	// 1. 创建查询
	matchAllQuery := elastic.NewMatchAllQuery()

	// 2. 设置分页条件
	from := 0
	size := 3

	// 3. 执行搜索
	result, err := client.GetESClient().Search().
		Index("hotel").
		Query(matchAllQuery).
		From(from).
		Size(size).
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 4. 处理结果
	data = make(map[string]interface{})
	data["total"] = result.TotalHits()
	data["took"] = result.TookInMillis

	var hits []map[string]interface{}
	for _, hit := range result.Hits.Hits {
		var docData map[string]interface{}
		if err := json.Unmarshal(hit.Source, &docData); err != nil {
			continue
		}

		docData["_id"] = hit.Id
		docData["_score"] = hit.Score
		hits = append(hits, docData)
	}

	data["hits"] = hits

	return data, nil
}
```

#### Golang测试代码

```go
func TestHotelSearchPaging(t *testing.T) {
	data, err := HotelSearchPaging(context.Background())
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

### 4.3 高亮

```json
GET /hotel/_search
{
  "query": {
    "match": {
      "name": "北京"
    }
  },
  "highlight": {
    "fields": {
      "name": {
        "require_field_match": "false"
      }
    }
  }
}
```

#### 对应的Golang操作

```go
func HotelSearchHighlight(ctx context.Context) (data map[string]interface{}, err error) {
	// 1. 创建查询
	matchAllQuery := elastic.NewMatchQuery("name", "北京")

	// 2. 设置高亮条件
	highlight := elastic.NewHighlight().
		Field("name").
		RequireFieldMatch(false)

	// 3. 执行搜索
	result, err := client.GetESClient().Search().
		Index("hotel").
		Query(matchAllQuery).
		Highlight(highlight).
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 4. 处理结果
	data = make(map[string]interface{})
	data["total"] = result.TotalHits()
	data["took"] = result.TookInMillis

	var hits []map[string]interface{}
	for _, hit := range result.Hits.Hits {
		var docData map[string]interface{}
		if err := json.Unmarshal(hit.Source, &docData); err != nil {
			continue
		}

		docData["_id"] = hit.Id
		docData["_score"] = hit.Score

		// 提取高亮内容
		if hit.Highlight != nil {
			if highlights, ok := hit.Highlight["name"]; ok && len(highlights) > 0 {
				docData["name"] = highlights[0] // 获取高亮的第一个片段
			}
		}

		hits = append(hits, docData)
	}
	data["hits"] = hits

	return data, nil
}
```

#### Golang测试代码

```go
func TestHotelSearchHighlight(t *testing.T) {
	data, err := HotelSearchHighlight(context.Background())
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

### 4.4 聚合查询

#### 桶聚合
```json
GET /hotel/_search
{
  "query": {
    "range": {
      "price": {
        "lte": 800
      }
    }
  },
  "size": 0,
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "order": {
          "_count": "asc"
        }, 
        "size": 10
      }
    }
  }
}
```

#### 对应的Golang操作

```go
func HotelSearchBucketAggregation(ctx context.Context) (data map[string]interface{}, err error) {
	// 1. 创建范围查询
	rangeQuery := elastic.NewRangeQuery("price").Lte(800)

	// 2. 创建聚合查询
	termsAggregation := elastic.NewTermsAggregation().
		Field("brand").
		OrderByCountAsc(). // 按文档数量升序排序
		Size(10)           // 返回前 10 个品牌

	// 3. 执行搜索
	result, err := client.GetESClient().Search().
		Index("hotel").
		Query(rangeQuery).
		Size(0). // 不需要返回文档，仅返回聚合结果
		Aggregation("brandAgg", termsAggregation).
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 4. 处理结果
	data = make(map[string]interface{})
	data["took"] = result.TookInMillis

	// 解析聚合结果
	brandAgg, found := result.Aggregations.Terms("brandAgg")
	if !found {
		return nil, fmt.Errorf("aggregation 'brandAgg' not found")
	}

	var brands []map[string]interface{}
	for _, bucket := range brandAgg.Buckets {
		brands = append(brands, map[string]interface{}{
			"brand": bucket.Key,
			"count": bucket.DocCount,
		})
	}

	data["brands"] = brands

	return data, nil
}
```

#### Golang测试代码

```go
func TestHotelSearchAggregation(t *testing.T) {
	data, err := HotelSearchBucketAggregation(context.Background())
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

#### 嵌套聚合

```json
GET /hotel/_search
{
  "size": 0,
  "aggs": {
    "brandAgg": {
      "terms": {
        "field": "brand",
        "order": {
          "scoreAgg.avg": "desc"
        },
        "size": 20
      },
      "aggs": {  
        "scoreAgg": {
          "stats": {
            "field": "score"
          }
        }
      }
    }
  }
}
```

#### 对应的Golang操作

```go
func HotelSearchNestedAggregation(ctx context.Context) (data map[string]interface{}, err error) {
	// 1. 创建聚合查询
	statsAggregation := elastic.NewStatsAggregation().Field("score") // 创建子聚合：统计 score 的信息

	termsAggregation := elastic.NewTermsAggregation().
		Field("brand").
		Order("scoreAgg.avg", false).                // 按 scoreAgg 的 avg 字段降序排序
		Size(20).                                    // 返回前 20 个品牌
		SubAggregation("scoreAgg", statsAggregation) // 添加子聚合

	// 2. 执行搜索
	result, err := client.GetESClient().Search().
		Index("hotel").
		Size(0). // 不返回文档，仅返回聚合结果
		Aggregation("brandAgg", termsAggregation).
		Do(ctx)

	if err != nil {
		return nil, err
	}

	// 3. 处理结果
	data = make(map[string]interface{})
	data["took"] = result.TookInMillis

	// 解析主聚合结果
	brandAgg, found := result.Aggregations.Terms("brandAgg")
	if !found {
		return nil, fmt.Errorf("aggregation 'brandAgg' not found")
	}

	var brands []map[string]interface{}
	for _, bucket := range brandAgg.Buckets {
		// 解析子聚合结果
		scoreAgg, found := bucket.Stats("scoreAgg")
		if !found {
			continue
		}

		brands = append(brands, map[string]interface{}{
			"brand":       bucket.Key,
			"count":       bucket.DocCount,
			"score_avg":   *scoreAgg.Avg,
			"score_min":   *scoreAgg.Min,
			"score_max":   *scoreAgg.Max,
			"score_sum":   *scoreAgg.Sum,
			"score_count": scoreAgg.Count,
		})
	}

	data["brands"] = brands

	return data, nil
}

```

#### Golang测试代码

```go
func TestHotelSearchNestedAggregation(t *testing.T) {
	data, err := HotelSearchNestedAggregation(context.Background())
	require.NoError(t, err)
	require.NotNil(t, data)
	t.Log(data)
}
```

## 5. 自动补全

### 5.1 自定义拼音分词器
```json
PUT /test
{
  "settings": {
    "analysis": {
      "analyzer": { 
        "my_analyzer": {
          "tokenizer": "ik_max_word",
          "filter": "py"
        }
      },
      "filter": {  
        "py": {  
          "type": "pinyin",
          "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true,
          "none_chinese_pinyin_tokenize": false
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "my_analyzer",
        "search_analyzer": "ik_smart"
      }
    }
  }
}
```

### 5.2 自动补全查询

```json
PUT /hotel
{
  "mappings": {
    "properties": {
      "name": {
        "type": "completion"
      }
    }
  }
}

POST /hotel/_bulk
{"index":{}}
{"name":"Hilton Hotel"}
{"index":{}}
{"name":"Holiday Inn"}
{"index":{}}
{"name":"Marriott Hotel"}
{"index":{}}
{"name":"Hyatt Regency"}
{"index":{}}
{"name":"The Ritz-Carlton"}


GET /hotel/_search
{
  "suggest": {
    "name_suggest": {
      "text": "h",
      "completion": {
        "field": "name",
        "skip_duplicates": "true",
        "size": 10
      }
    }
  }
}
```

#### 对应的Golang操作

```go
package main

import (
	"context"
	client2 "esdemo/client"
	"github.com/gin-contrib/cors"
	"net/http"

	"github.com/gin-gonic/gin"
	"github.com/olivere/elastic/v7"
)

var client *elastic.Client

func autocompleteHandler(c *gin.Context) {
	client = client2.GetESClient()
	query := c.Query("q")
	if query == "" {
		c.JSON(http.StatusBadRequest, gin.H{"error": "query parameter 'q' is required"})
		return
	}

	// 构造自动补全查询
	suggest := elastic.NewCompletionSuggester("name-suggest").
		Prefix(query).
		Field("name").
		Size(10)

	searchResult, err := client.Search().
		Index("hotel").
		Suggester(suggest).
		Do(context.Background())

	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
		return
	}

	// 解析补全建议结果
	var suggestions []string
	if suggestResults, found := searchResult.Suggest["name-suggest"]; found {
		for _, entry := range suggestResults {
			for _, option := range entry.Options {
				suggestions = append(suggestions, option.Text)
			}
		}
	}

	c.JSON(http.StatusOK, gin.H{"suggestions": suggestions})
}

func main() {
	r := gin.Default()
	r.Use(cors.Default())
	r.GET("/autocomplete", autocompleteHandler)
	err := r.Run(":8080")
	if err != nil {
		return
	} // 启动服务，监听 8080 端口
}
```

#### 实现的效果

![image-20241127134540602](https://lyb-1305354270.cos.ap-beijing.myqcloud.com/lhcos-data/image-20241127134540602.png)
