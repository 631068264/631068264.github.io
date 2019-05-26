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

![](http://ww4.sinaimg.cn/large/006tNc79ly1g3dwhn16i7j31pw0fq0tm.jpg)

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

![](http://ww4.sinaimg.cn/large/006tNc79ly1g3dwrz6ovzj31pi0luq45.jpg)

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

ERROR 
```
sqlalchemy.exc.OperationalError: (_mysql_exceptions.OperationalError) (1366, "Incorrect integer value: 'age + VALUES(age)' for column 'age' at row 1")
[SQL: INSERT INTO user (name, age) VALUES (%s, %s) ON DUPLICATE KEY UPDATE age = %s]
[parameters: ('bulko11', 11, 'age + VALUES(age)')]
```

# all() first() scalar()

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

# 常用

## sqlalchemy object to list of dict

```python
def sqlalchemy2dict(result: typing.List[_LW]) -> Results:
    return [r._asdict() for r in result]
```

## 聚合

```python
def get_wheres(query: Query) -> Query:
    query = query.filter(
        ((cls.src_ip == ip) | (cls.dst_ip == ip))
    )
    if kid is not None:
        query = query.filter(cls.kid == kid)

    return query.filter((start_ts <= cls.time) & (cls.time <= end_ts))

# count
select = session.query(func.count(
    distinct(cls.signature_id)).label('count'))
count = get_wheres(select).scalar()

# query
select = session.query(
    func.group_concat(distinct(cls.src_ip)).label('src_ip'),
    func.group_concat(distinct(cls.dst_ip)).label('dst_ip'),
    func.any_value(cls.signature_id).label('signature_id'),
    func.any_value(cls.level).label('level'),
    func.any_value(cls.signature).label('signature'),
    func.any_value(cls.category).label('category'),
    func.any_value(cls.kid).label('kid'),
    func.sum(cls.count).label('count')
)
query = get_wheres(select).group_by(cls.signature_id).offset(
    (page - 1) * page_size).limit(page_size).all()

return count, sqlutil.sqlalchemy2dict(query)
```

```python
query = session.query(
            cls.sensor_id.label('sensorid'),
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
for i in ip_range_list:
    ip_range = i[InnerIpRange.ip_range.key]
    min_ip, max_ip = int(min(ip_range)), int(max(ip_range))
    or_list.append(
        and_(min_ip <= InnerMachine.ip4, InnerMachine.ip4 <= max_ip))

query = query.filter(or_(*or_list))
```

## join

```python
def get_wheres(query: Query) -> Query:
    query = query.join(InnerMachine, cls.inner_machine_id == InnerMachine.id)
    query = query.filter(InnerMachine.ip == ip)
    return query

# 列表
select = session.query(cls.inner_machine_id, (cls.ignore_time > cls.update_time).label('ignore'))
query = get_wheres(select).limit(1)

c = query.first()
if c is None:
    return False, False
return True, c.ignore
```

## 模糊查询

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