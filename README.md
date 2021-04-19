# neo4j-study

[sanbox](https://sandbox.neo4j.com/)
### 基础概念
- 点 
- 边

### demo
- Match 
    - `Match (v: Movie) where v.released > 2000 RETURN v limit 5`  // Match(m: VertexType)   查询 2000后 放映的电影
    - `Match (v: Movie) where v.released < 2000 RETURN count(v)`  查询 2000前 放映的电影的总数
    - `Match (p: Person)-[d: DIRECTED]->(m: Movie) Where m.released > 2000 RETURN p,d,m`  查询2000后所有导演指导的电影关系
    - `match(m: Movie)-[a:ACTED_IN]-(p:Person) where m.released > 2000 return m,a,p` 查询2000后 电影与参与演员的关系
    - `match (n) return count(n)` 查询当前空间重所有 vertex的数量
    - `match (m: Movie) return m.title, m.released` 查询movie具体字段
    - `Match (p:Person {name: 'Tom Hanks'}) RETURN p` 条件查询与下等同  
        - `Match (p:Person) where p.name = 'Tom Hanks' RETURN p`
        - `Match (m:Movie {name: 'Cloud Atlas'}) where m.released > 2010 and m.released < 2015  RETURN m` 
    - `MATCH (tom:Person {name: "Tom Hanks"})-[:ACTED_IN]->(:Movie)<-[:ACTED_IN]-(p:Person) return p.name` 寻找与 Tom Hanks 合作过电影的所有人
    - `MATCH (p:Person {name: 'Kevin Bacon'})-[*1..3]-(hollywood) return DISTINCT p, hollywood`  查询与 Person{name: "Kevin Bacon"} 有关系的3跳路径
- 画点
    - `create (p: Person {name: 'john Doe'}) return p` 创建一个 person类型的vertex
- MERGE 组合
    - 实现UPSERT
    
    ``` 
        MERGE (p:Person {name: 'John Doe'})
        ON MATCH SET p.lastLoggedInAt = timestamp()
        ON CREATE SET p.createdAt = timestamp()
        Return p
    
        如果Person 不存在就 SET p.createdAt = timestamp()
        如果存在 就 SET p.lastLoggedInAt = timestamp()
    ``` 
  
- 画线

找到 Person.name = Tom Hanks and Movie.title = "Cloud Atlas" 的两个点
创建一个条WATCHED线 链接它们
``` 
MATCH (p: Person), (m: Movie) where p.name = "Tom Hanks" and m.title = "Cloud Atlas"
Create (p)-[w:WATCHED]->(m)
RETURN type(w)
```
    
    
### Execute

```go
func Execute(command string, params map[string]interface{}) ([]map[string]interface{}, error) {
	session := Ne04JDriver.NewSession(neo4j.SessionConfig{AccessMode: neo4j.AccessModeWrite})
	defer session.Close()

	greeting, err := session.WriteTransaction(func(transaction neo4j.Transaction) (interface{}, error) {
		result, err := transaction.Run(
			command, params)
		if err != nil {
			return nil, err
		}

		keys, err := result.Keys()
		if err != nil {
			return nil, err
		}

		var resp []map[string]interface{}
		for result.Next() {
			item := map[string]interface{}{}
			for k, v := range keys {
				i := result.Record().Values[k]
				item[v] = i
			}

			resp = append(resp, item)
		}

		return resp, result.Err()
	})

	if err != nil {
		return nil, errors.WithStack(err)
	}

	return greeting.([]map[string]interface{}), nil
}
```

### bind
``` 
// Initialize logger
var Ne04JDriver neo4j.Driver

func InitNeo4J() error {
	driver, err := neo4j.NewDriver(fmt.Sprintf("neo4j://%s:%d", address, port), neo4j.BasicAuth(username, password, ""))
	if err != nil {
		return errors.WithStack(err)
	}

	Ne04JDriver = driver

	return nil
}

func Execute(command string, params map[string]interface{}) ([]map[string]interface{}, error) {
	session := Ne04JDriver.NewSession(neo4j.SessionConfig{AccessMode: neo4j.AccessModeWrite})
	defer session.Close()

	greeting, err := session.WriteTransaction(func(transaction neo4j.Transaction) (interface{}, error) {
		result, err := transaction.Run(
			command, params)
		if err != nil {
			return nil, err
		}

		keys, err := result.Keys()
		if err != nil {
			return nil, err
		}

		var resp []map[string]interface{}
		for result.Next() {
			item := map[string]interface{}{}
			for k, v := range keys {
				i := result.Record().Values[k]
				item[v] = i
			}

			resp = append(resp, item)
		}

		return resp, result.Err()
	})

	if err != nil {
		return nil, errors.WithStack(err)
	}

	return greeting.([]map[string]interface{}), nil
}

// list 绑定
func BindAllEdges(resultSet []map[string]interface{}, v interface{}) error {
	refType := reflect.TypeOf(v)
	refVal := reflect.ValueOf(v)
	if refType.Kind() != reflect.Ptr {
		return errors.New("类型错误 应该为&[]")
	}

	// 解引用看内部类型
	if refType.Elem().Kind() != reflect.Slice {
		return errors.New("类型错误 应该为&[]")
	}

	elem := refType.Elem().Elem() // 内部具体的类型
	sliceVal := refVal.Elem()     // 具体slice

	// 建立一个新数组
	newArr := make([]reflect.Value, 0)

	// 建立一个item
	for idx := range resultSet {
		index := idx
		edge, err := newEdge(resultSet[index], elem)
		if err != nil {
			return errors.WithStack(err)
		}

		newArr = append(newArr, edge.Elem())
	}

	// 重写
	resArr := reflect.Append(sliceVal, newArr...)
	sliceVal.Set(resArr)

	return nil
}

// 无中生有
func newEdge(record map[string]interface{}, elemType reflect.Type) (resVal reflect.Value, err error) {
	defer func() {
		if error := recover(); error != nil {
			err = errors.Errorf("%s", error)
		}
	}()
	// 1. 获取struct所有的 json tag
	// 2. 对应ColNames
	// 3. 类型转换
	// 4. 填数据
	refVal := reflect.New(elemType)
	refTypeElem := elemType
	refValElem := refVal.Elem()

	for i := 0; i < refTypeElem.NumField(); i++ {
		typeElem := refTypeElem.Field(i)
		structElem := refValElem.Field(i)
		if !structElem.CanSet() {
			continue
		}

		// 取tag
		tag := typeElem.Tag.Get("json")
		if tag == "" {
			tag = typeElem.Name
		}

		if tag == "-" {
			continue
		}

		col, ex := record[tag]
		if !ex {
			continue
		}

		switch typeElem.Type.Kind() {
		case reflect.Float64:
			f, ex := col.(float64)
			if !ex {
				continue
			}
			structElem.SetFloat(f)
		case reflect.String:
			s, ex := col.(string)
			if !ex {
				continue
			}
			s = strings.TrimSpace(s)
			s = strings.Replace(s, "\"", "", -1)
			structElem.SetString(s)
		}
	}

	return refVal, err
}
```