# 预编译语句

自 3.2 起，StarRocks 提供预编译语句（prepared statement），用于多次执行结构相同仅部分变量不同的 SQL 语句，能显著提升执行效率并且还能防止 SQL 注入。

## 功能

预编译语句的主要工作流程如下：

1. 预编译：准备SQL 语句。语句中的变量用占位符 `?` 表示。FE 解析语句并生成执行计划。
2. 执行：声明变量后，将变量传递至语句中并执行语句。并且可以使用不同的变量多次执行该语句。

**优势以及应用场景**

- 节省解析语句的开销。业务中应用程序通常会使用不同变量多次执行结构相同的语句。使用预编译语句后，StarRocks 只需要在预编译阶段解析一次语句，后续多次执行相同语句（变量不同）时直接使用解析结果，因此能显著提高语句（特别是复杂语句）执行性能。
- 防止 SQL 注入攻击。分离了语句和变量，将用户输入的数据作为参数进行传递，而不是将其直接拼接到语句中。这样可以防止恶意用户利用输入来执行恶意的 SQL 代码。

**使用说明**

预编译语句仅在当前 session 中生效，不适用于其他 session。当前 session 退出后，所创建预编译语句自动被删除。

## 语法

预编译语句如下：

- PREPARE：准备语句，语句中变量用占位符 `?` 表示。
- SET：声明语句中的变量。
- EXECUTE：传递已声明的变量并执行语句。
- DROP PREPARE 或 DEALLOCATE PREPARE：删除预编译语句。

### PREPARE

**语法：**

```SQL
PREPARE stmt_name FROM preparable_stmt
```

**参数说明：**

- `stmt_name`：预编译语句名称。
- `preparable_stmt`：预处理语句 SQL。其中的变量占位符为英文半角问号 (`?`)。

**示例：**

准备一个 `INSERT INTO VALUES()`语 句，其中具体的值用 `?` 占位。

```SQL
PREPARE insert_stmt FROM 'INSERT INTO users (id, country, city, revenue) VALUES (?, ?, ?, ?)';
```

### SET

**语法：**

```Plain
SET @var_name = expr [, ...];
```

**参数说明：**

- `var_name`：自定义变量名称。
- `expr`：自定义变量。

**示例：**

声明四个变量。

```SQL
SET @id1 = 1, @country1 = 'USA', @city1 = 'New York', @revenue1 = 1000000;
```

详细信息，请参见[用户自定义变量](./user_defined_variables.md)。

### EXECUTE

**语法：**

```SQL
EXECUTE stmt_name [USING @var_name [, @var_name] ...]
```

**参数说明：**

- `var_name`：在 `SET` 语句中已声明的变量名称。
- `stmt_name`：预编译语句名称。

**示例：**

传递变量并执行 `INSERT` 语句。

```SQL
EXECUTE insert_stmt USING @id1, @country1, @city1, @revenue1;
```

### DEALLOCATE PREPARE 或 DROP PREPARE

```SQL
{DEALLOCATE | DROP} PREPARE stmt_name
```

**参数说明：**

`stmt_name`：预编译语句名称。

**示例：**

删除一个预编译语句。

```SQL
DROP PREPARE insert_stmt;
```

## 示例

### 使用预编译语句

本示例介绍如何使用预编译语句增删改查 StarRocks 表的数据。

假设已经创建如下数据库 demo 和表 users。

```SQL
CREATE DATABASE IF NOT EXISTS demo;
USE demo;
create table users (
  id bigint not null,
  country string,
  city string,
  revenue bigint
)
primary key (id)
distributed by hash(id);
```

1. 准备语句以供执行。

   ```SQL
   PREPARE insert_stmt FROM 'INSERT INTO users (id, country, city, revenue) VALUES (?, ?, ?, ?)';
   PREPARE select_all_stmt FROM 'SELECT * FROM users';
   PREPARE select_by_id_stmt FROM 'SELECT * FROM users WHERE id = ?';
   PREPARE update_stmt FROM 'UPDATE users SET revenue = ? WHERE id = ?';
   PREPARE delete_stmt FROM 'DELETE FROM users WHERE id = ?';
   ```

2. 声明语句中的变量。

   ```SQL
   SET @id1 = 1, @id2 = 2;
   SET @country1 = 'USA', @country2 = 'Canada';
   SET @city1 = 'New York', @city2 = 'Toronto';
   SET @revenue1 = 1000000, @revenue2 = 1500000, @revenue3 = (select (revenue)*1.1 from users);
   ```

3. 使用已声明的变量来执行语句。

   ```SQL
   -- 新增两条数据
   EXECUTE insert_stmt USING @id1, @country1, @city1, @revenue1;
   EXECUTE insert_stmt USING @id2, @country2, @city2, @revenue2;
   
   -- 查询表中所有数据
   EXECUTE select_all_stmt;
   
   -- 分别查询 ID 为 1 和 2 的数据
   EXECUTE select_by_id_stmt USING @id1;
   EXECUTE select_by_id_stmt USING @id2;
   
   -- 对ID 为1的数据行进行部分更新，仅更新其列 revenue
   EXECUTE update_stmt USING @revenue3, @id1;
   
   -- 删除 ID 为 1 的数据
   EXECUTE delete_stmt USING @id1;
   ```

4. 删除预编译语句。

   ```SQL
   DROP PREPARE insert_stmt;
   DROP PREPARE select_all_stmt;
   DROP PREPARE select_by_id_stmt;
   DROP PREPARE update_stmt;
   DROP PREPARE delete_stmt;
   ```

### JAVA 应用程序使用预编译语句

本示例介绍 JAVA 应用程序如何通过 JDBC 驱动使用预编译语句增删改查 StarRocks 表的数据。

1. 在 JDBC URL 中配置 StarRocks 的连接地址时，需要启用服务端预编译语句。

    ```Plain
    jdbc:mysql://<fe_ip>:<fe_query_port>/useServerPrepStmts=true
    ```

2. StarRocks Github 项目中提供 [JAVA 代码示例](https://github.com/StarRocks/starrocks/blob/main/fe/fe-core/src/test/java/com/starrocks/analysis/PreparedStmtTest.java)，说明如何增删改查 StarRocks 表的数据。
