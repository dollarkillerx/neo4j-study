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


    