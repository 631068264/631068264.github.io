---
layout:     post
rewards: false
title:      sqlalchemy 坑
categories:
    - sqlalchemy
---

# model 必须要主键 

<span class='heimu'>在我看来ORM底层就好好的拼字符串sql搞什么奇葩设定 </span>

[How do I map a table that has no primary key](https://docs.sqlalchemy.org/en/13/faq/ormconfiguration.html#how-do-i-map-a-table-that-has-no-primary-key)

大多数ORM要求对象定义某种主键，因为内存中的对象必须对应于数据库表中唯一可识别的行; 至少，这允许对象可以作为UPDATE和DELETE语句的目标，
这些语句将仅影响该对象的行而不影响其他行。但是，主键的重要性远不止于此。

在SQLAlchemy中，所有ORM映射对象始终Session 使用称为身份映射的模式在其特定数据库行中唯一链接，
该模式是SQLAlchemy使用的工作单元系统的核心，也是最关键的模式。 ORM使用的常见（而不是那么常见）模式。


# insert

```sql
CREATE TABLE `user` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(120) NOT NULL,
  `age` int(10) unsigned NOT NULL DEFAULT '0',
  `ts` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
)
```

model
```python
class User(BaseModel):
    __tablename__ = 'user'
    id = Column(INTEGER(unsigned=True), primary_key=True)
    name = Column(VARCHAR(120))
    age = Column(INTEGER(unsigned=True))
    ts = Column(TIMESTAMP())
```


## add
 
<span class='heimu'>一股傻逼气息扑面而来</span> Cannot insert NULL value in
column, but I have a default value specified
```python
session.add(User(name='faf'))
```

ERROR 
```error
sqlalchemy.exc.OperationalError: (_mysql_exceptions.OperationalError) (1048, "Column 'age' cannot be null")
```

执行的是这个sql <span class='heimu'>坑爹啊</span>
```
INSERT INTO user (name, age, ts) VALUES (%s, %s, %s)
('faf', None, None)
```


## speed up
[sqlalchemyのinsert高速化](https://qiita.com/dekisugikun/items/068f4de5686e10b079b2)

```python
session.execute(User.__table__.insert(), {'name': 'affa'})

# 批量insert
session.bulk_save_objects([User(name='bulko') for i in range(0, 5)])

# 推荐写法
data = [{'name': 'bulk', 'age': i} for i in range(0, 5)]
session.execute(User.__table__.insert(), data)
```

```sql
INSERT INTO user (name) VALUES (%s)
('affa',)

INSERT INTO user (name, age) VALUES (%s, %s)
(('bulk', 0), ('bulk', 1), ('bulk', 2), ('bulk', 3), ('bulk', 4))
```

# on_duplicate_key_update

**only for mysql** [insert-on-duplicate-key-update-upsert](https://docs.sqlalchemy.org/en/13/dialects/mysql.html#insert-on-duplicate-key-update-upsert)

![](https://cdn.jsdelivr.net/gh/631068264/img/006tNc79ly1g3dwhn16i7j31pw0fq0tm.jpg)

**update or insert by unique key**

```python
from sqlalchemy.dialects.mysql import insert, Insert

def upsert(model: declarative_base, data: typing.Union[typing.Dict, typing.List[typing.Dict]],
           update_field: typing.List) -> Insert:
    """
    on_duplicate_key_update for mysql
    """
    # https://docs.sqlalchemy.org/en/13/dialects/mysql.html#insert-on-duplicate-key-update-upsert
    stmt = insert(model).values(data)
    d = {f: getattr(stmt.inserted, f) for f in update_field}
    return stmt.on_duplicate_key_update(**d)

```

```python
data = [dict(name=f'bulko{i}', age=i) for i in range(0, 10)]
stmt = upsert(User, data, update_field=list(data[0].keys()))

session.execute(stmt)
session.commit()
```

![](https://cdn.jsdelivr.net/gh/631068264/img/006tNc79ly1g3dwrz6ovzj31pi0luq45.jpg)

```sql
INSERT INTO user (name, age) VALUES (%s, %s), (%s, %s), (%s, %s), (%s, %s), (%s, %s), (%s, %s), (%s, %s), (%s, %s), (%s, %s), (%s, %s) ON DUPLICATE KEY UPDATE name = VALUES(name), age = VALUES(age)
('bulko0', 0, 'bulko1', 1, 'bulko2', 2, 'bulko3', 3, 'bulko4', 4, 'bulko5', 5, 'bulko6', 6, 'bulko7', 7, 'bulko8', 8, 'bulko9', 9)
```

要想再复杂点<span class='heimu'>马上就跪了</span>

```python
i = 11
data = dict(name=f'bulko{i}', age=i)
stmt = insert(User).values(data)
stmt = stmt.on_duplicate_key_update({
    'age': 'age + VALUES(age)',
})

session.execute(stmt)
session.commit()
```

ERROR 
```
sqlalchemy.exc.OperationalError: (_mysql_exceptions.OperationalError) (1366, "Incorrect integer value: 'age + VALUES(age)' for column 'age' at row 1")
[SQL: INSERT INTO user (name, age) VALUES (%s, %s) ON DUPLICATE KEY UPDATE age = %s]
[parameters: ('bulko11', 11, 'age + VALUES(age)')]
```

然后只能 先select 在update <span class='heimu'>超傻逼</span>
```python
for d in data:
    inner_machine_id = d['inner_machine_id']
    risk_machine = RiskInnerMachine.get_by_inner_machine_id(db, inner_machine_id)
    if risk_machine:
        RiskInnerMachine.update_by_inner_machine_id(db, inner_machine_id, {
            RiskInnerMachine.is_risk: risk_machine.is_risk if risk_machine.is_risk == 1 else d['is_risk'],
            RiskInnerMachine.threat_access: risk_machine.threat_access + d['threat_access'],
            RiskInnerMachine.risk_file: risk_machine.risk_file + d['risk_file'],
            RiskInnerMachine.attack_event: risk_machine.attack_event + d['attack_event'],
            RiskInnerMachine.access_rule: risk_machine.access_rule + d['access_rule'],
            RiskInnerMachine.update_time: d['update_time'],
        })
    else:
        d['fall_type'] = 0
        d['ignore_time'] = 0
        db.add(RiskInnerMachine(**d))
```

## 使用 compiles

[Custom SQL Constructs and Compilation Extension](https://docs.sqlalchemy.org/en/13/core/compiler.html?highlight=compiler)
[SQLAlchemy ON DUPLICATE KEY UPDATE](https://stackoverflow.com/a/10561643/5360312)

```python
from sqlalchemy.ext.compiler import compiles

# 这个import很关键
from sqlalchemy.sql.expression import Insert

@compiles(Insert, 'mysql')
def on_duplicate_key_update(insert, compiler, **kw):
    def _gen_fv_dict(fv):
        sql = []
        if isinstance(fv, dict):
            for f, v in fv.items():
                sql.append(f' {f} = {v} ')

        elif isinstance(fv, list):
            for f in fv:
                sql.append(f' {f} = VALUES({f}) ')
        return ','.join(sql)

    s = compiler.visit_insert(insert, **kw)
    if 'on_duplicate_key_update' in insert.kwargs:
        return s + ' ON DUPLICATE KEY UPDATE ' + _gen_fv_dict(insert.kwargs['on_duplicate_key_update'])
    return s
```
效果拔群，但是会拖慢正常的insert,拖慢很多

**没用**`compiles`插1000个，用0.138sec ，**用了**0.624sec 1000个
0.540一个。。。。

test code 
```python
class User(BaseModel):
    __tablename__ = 'user'
    id = Column(INTEGER(unsigned=True), primary_key=True)
    name = Column(VARCHAR(120))
    age = Column(INTEGER(unsigned=True))
    
data = [{'name': f'bulk{i}', 'age': i} for i in range(0, 1000)]


def test1():
    # 0.138 0.624  0.540
    with mysql() as db:
        db.execute(User.__table__.insert(), data[0])
        db.commit()


def test2():
    # 0.253 0.731
    with mysql() as db:
        db.execute(insert(User).values(data))
        db.commit()


def test3():
    with mysql() as db:
        db.add_all(data)
        db.commit()


import profile

profile.run('test1()')
# profile.run('test2()')
# profile.run('test3()')
```


# all() & first() & scalar()

只要一个object直接`first()` 多个用`all()` 数字用
```python
# SELECT user.name AS user_name, user.age AS user_age FROM user          return <class 'list'>:[....]
user = session.query(User.name, User.age).all()

# SELECT user.name AS user_name, user.age AS user_age FROM user LIMIT 1  return  ('bulko0', 0)
user = session.query(User.name, User.age).first()

# SELECT user.name AS user_name, user.age AS user_age FROM user LIMIT 1  return  <class 'list'>: [('bulko0', 0)]
user = session.query(User.name, User.age).limit(1).all()

# SELECT user.name AS user_name, user.age AS user_age FROM user LIMIT 1  return  ('bulko0', 0)
user = session.query(User.name, User.age).limit(1).first()

# SELECT user.name AS user_name, user.age AS user_age FROM user LIMIT 1  return  'bulko0'
user = session.query(User.name, User.age).limit(1).scalar()
```

找不到目标
```
user_id = 100
# []
user = session.query(User.name, User.age).filter(User.id == user_id).all()
# None
user = session.query(User.name, User.age).filter(User.id == user_id).first()
user = session.query(User.name, User.age).filter(User.id == user_id).limit(1).all()
user = session.query(User.name, User.age).filter(User.id == user_id).limit(1).first()
user = session.query(User.name, User.age).filter(User.id == user_id).limit(1).scalar()
# 0
user = session.query(func.count('*')).filter(User.id == user_id).scalar()
# (0,)
user = session.query(func.count('*')).filter(User.id == user_id).first()
```

# 常用gist

## sqlalchemy object to list of dict

```python
def sqlalchemy2dict(result: typing.List[_LW]) -> Results:
    return [r._asdict() for r in result]
```

## 聚合

```python
# count
select = session.query(func.count(
    distinct(cls.id)).label('count'))
count = get_wheres(select).scalar()

# query
select = session.query(
    func.group_concat(distinct(cls.aa)).label('aa'),
    func.any_value(cls.id).label('id'),
    func.sum(cls.count).label('count')
)
query = get_wheres(select).group_by(cls.signature_id).offset(
    (page - 1) * page_size).limit(page_size).all()

return count, sqlutil.sqlalchemy2dict(query)
```

```python
query = session.query(
            # sum -> decimal to int
            func.sum(cls.count).op('div')(1).label('count'),
            func.Hour(func.FROM_UNIXTIME(cls.create_time)).label('hour'),
        ).filter(
            (start_time <= cls.create_time) & (cls.create_time <= end_time)
        ).group_by(
            'sensorid', 'hour'  # 自定义label
        ).order_by('hour')
        return query.all()
```

## and or 拼接

```python
or_list = []
for ip_range in ip_range_list:
    min_ip, max_ip = int(min(ip_range)), int(max(ip_range))
    or_list.append(and_(min_ip <= xx.ip4, xx.ip4 <= max_ip))

query = query.filter(or_(*or_list))
```

## join

```python
def get_wheres(query: Query) -> Query:
    query = query.join(xx, cls.id == xx.id)
    query = query.filter(xx.ip == ip)
    return query

# 列表
select = session.query(cls.id, (cls.c > cls.a).label('ignore'))
query = get_wheres(select).limit(1)

c = query.first()
if c is None:
    return False, False
return True, c.ignore
```

## 子查询 .subquery()

```python
sub = session.query(func.any_value(stat_column).label('name')) \
    .join(xx, xx.id == cls.id) \
    .filter((start_ts <= xx.date) & (xx.date <= end_ts)) \
    .group_by(cls.md5).subquery()

query = session.query('name', func.count('*').label('count')) \
    .select_from(sub) \
    .group_by('name') \
    .order_by(desc('count'))
```

## 模糊查询
可以用 mysql concat `query =
query.filter(func.concat(*columns).like(f'%{keyword}%'))` 将字段连起来查询

但是会有造成以下情况 **kw 关键字 f 字段**

kw = ab f1 = a f2=b 是匹配的 ,当然f1
f2中间可以塞其他特殊字符作为**连接符**，但是很难保证kw输入什么。


关于转义

在mysql中，反斜杠在字符串中是转义字符，在进行语法解析时会进行**一次转义**，所以当我们在insert字符时，insert `\\`
在数据库中最终只会存储`\`。

而在mysql的like语法中，like后边的字符串除了会在语法解析时转义一次外，还会在正则匹配时进行**第二次的转义**。
因此如果期望最终匹配到`\`，就要反转义两次，也即由`\`到`\\`再到`\\\\`。


```python
def search_keyword(query: Query, keyword: str, columns: Columns) -> Query:
    if keyword:
        # 路经关键字查询反斜杠转义
        keyword = keyword.replace('\\', '\\\\')

        if isinstance(columns, typing.Sequence):
            or_list = []
            for c in columns:
                or_list.append(c.like(f'%{keyword}%'))
            query = query.filter(or_(*or_list))
        else:
            query = query.filter(columns.like(f'%{keyword}%'))
    return query
```

## 动态filter

```python
class FILTER_OP(object):
    EQ = "="
    NE = "!="
    GT = ">"
    LT = "<"
    LIKE = "like"
    NOT_LIKE = "not like"
    # https://docs.sqlalchemy.org/en/13/orm/internals.html#sqlalchemy.orm.properties.ColumnProperty.Comparator
    NAME_DICT = {
        EQ: 'eq',
        NE: 'ne',
        GT: 'gt',
        LT: 'lt',
        LIKE: 'like',
        NOT_LIKE: 'notlike',
    }

    ALL = list(NAME_DICT.keys())

'''
raw_filter jsonschema 结构
    
'filter': {
            'type': 'array',
            'items': {
                'type': 'object',
                'required': ['key', 'op', 'value'],
                "properties": {
                    "key": {"type": "string", "enum": 条件字段},
                    "op": {"type": "string", "enum": const.FILTER_OP.ALL},
                    "value": {},
                }
            }
        },

'''


def sql_filter(query: Query, model: declarative_base, raw_filter: typing.List[typing.Dict]) -> Query:
    """
    动态filter and 连接
    const.FILTER_OP 控制操作
    """
    if raw_filter:
        for raw in raw_filter:
            key, op, value = raw['key'], const.FILTER_OP.NAME_DICT[raw['op']], raw['value']
            column = getattr(model, key)
            attr = list(filter(lambda e: hasattr(column, e % op), ['%s', '%s_', '__%s__']))[0] % op
            query = query.filter(getattr(column, attr)(value))

    return query
```


## 时间间隔统计

```python
query = session.query(
            func.count('*').label('count'),
            cls.timestamp.op('div')(duration).label('time')
        ).filter(
            (start_ts <= cls.timestamp) & (cls.timestamp <= end_ts)
        ).group_by('time').order_by('time')

        result = []
        record = query.all()
        for r in record:
            result.append({
                'timestamp': r.time * duration,
                'count': r.count,
            })
        return result
        
        
def fill_timestamp_count(start_ts, end_ts: int, interval: int, raw_result: Results) -> Results:
    """补全缺失的时间分布"""
    if not raw_result:
        return []
    begin = start_ts // interval * interval
    end = end_ts // interval * interval
    raw_result = {r['timestamp']: r for r in raw_result}
    result = {t: {'timestamp': t, 'count': 0} for t in range(begin, end + interval, interval)}

    for r in raw_result:
        result[r] = raw_result[r]

    return sorted(result.values(), key=itemgetter('timestamp'))
```

