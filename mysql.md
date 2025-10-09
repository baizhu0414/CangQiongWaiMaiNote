# 关键字对比和识别
> sum&count
> 事务控制
> 触发器
> 数据库语言分类
- sum和count的区别
    - count返回数量，不管值是什么
    - sum会将返回值相加
    ```sql
    <!-- 统计姓张的学生总人数 -->
    A. select sum(case when name like '张%' then 1 else 0 end) as zhang_first_name // 对数字求和
        from
        student_table;
    B. select count(case when name like '张%' then 2 else null end) as zhang_first_name // 计数，null不计数，2作为一个标志进行计数+1操作，不管值是多少
        from
        student_table;
    C. select sum(case when name like '张%' then 1 else null end) as zhang_first_name // null和0效果相同，除非所有结果都是null，则结果是null而不是0.
        from
        student_table;
    ```

- InnoDB引擎中的事务控制语句：
    - ROLLBACK：回滚会结束用户的事务，并撤销正在进行的所有未提交的修改。它只能作用于当前尚未提交的事务。
    - savepoint identifier：表示在事务中创建一个保存点，一个事务可以有多个保存点；
        - release savepoint identifier删除一个事务的保存点；
        - ROLLBACK TO savepoint identifier回滚到某个保存点。
    - commit：COMMIT是事务成功完成的标志，该事务对数据库所做的所有更改就会被持久化，一个已经结束的事务不能再被回滚。
    - 注意：MySQL中默认自动提交，使用begin/start transaction显示开启事务则自动提交暂时禁用，直到手动commit/rollback.
    
- 触发事件有哪些？
    - 触发器分类：
        - after触发器；
        - before触发器；
    - 触发事件：
        - update可访问NEW&OLD伪数据；
        - insert可访问NEW伪数据；
        - delete可访问OLD伪数据；
    - 伪记录：
        - NEW 代表新的数据行；
        - OLD 代表旧的数据行；

    ```SQL
    <!-- 分类：before insert/ after insert -->
    <!-- before insert触发器举例：在向employees表插入数据前检查员工年龄是否符合要求 -->
    <!-- DELIMITER修改分隔符防止'；'符号号误判 -->
    <!--insert可以使用 NEW 伪数据-->
    <!--BEGIN...END 包裹触发器执行逻辑-->
    <!-- SIGNAL：主动抛出错误的关键字，用于阻止不符合条件的操作。SQLSTATE '45000'：错误状态码，45000是 MySQL 预定义的 “用户自定义错误” 代码。SET MESSAGE_TEXT：设置错误提示信息，告知用户操作被阻止的原因。 -->
    DLIMITER //
    CREATE TRIGGER check_age_before_insert
    BEFORE INSERT ON employees 
    FOR EACH ROW
    BEGIN 
        IF timestamp(YEAR, NEW.birthdate, curdate())<18 THEN
            signal sqlstate '45000' set message_text='员工年龄必须至少18岁';
        END IF
    END //
    DELIMITER;

    <!-- after insert触发器.-->
    DLIMITER //
    CREATE TRIGGER ...
    AFTER INSERT ON xx
    FOR EACH ROW
    BEGIN
        update xx set xx=xxx where ww=www; <!-- update可以调用NEW&OLD伪数据 -->
    END //
    DELIMITER ;


    <!-- 分类：before update/ after update -->
    DLIMITER //
    CREATE TRIGGER ...
    BEFORE UPDATE ON xx
    FOR EACH ROW
    BEGIN
        set ...
    END //
    DELIMITER ;

    <!-- 分类：before delete/ after delete -->

    ```
        
- 数据库语言分类：DDL、DML、DCL、TCL
    - DDL数据定义语言：特点是无法回滚。
        - CREATE, ALTER, DROP、TRUNCATE（清空表的所有记录，先drop再create）、COMMENT（添加字段和表的描述）、RENAME；
            ```sql
            <!-- alter+comment -->
            ALTER TABLE users COMMENT '用户的有效邮箱，用于接收通知';
            <!-- alter+add：设置外键-->
            ALTER TABLE scores
            ADD CONSTRAINT FK_uid
            FOREIGN KEY (scores.uid) REFERENCES students (uid)

            <!-- 修改表：ALTER TABLE ,关键字column可省略-->
            1. change:列名和类型同时修改
            ALTER TABLE students
            CHANGE COLUMN id student_id BIGINT;

            2. modify：仅能修改类型
            ALTER TABLE students
            MODIFY COLUMN name VARCHAR(200);

            3. alter:修改默认值
            ALTER TABLE students
            ALTER COLUMN student_id SET DEFAULT 0;
            ```
    - DML数据操作语言：在事务中进行，支持commit提交或rollback回滚。
        - update/insert/delete/select
            ```sql
            <!-- case when使用：适合多重条件，结果是一个列 -->
            <!-- vs. IF-ELSE：只控制代码块执行流程，适合事务 -->
            1. 简单case表达式：
                CASE 列名 WHEN 值1 THEN 结果1 ELSE 默认值 END
            2. '搜索case表达式'：适用于复杂条件判断
                CASE WHEN 条件1 THEN 结果1 ELSE 默认值 END
            例子：
            <!--1. FROM-WHERE-GROUP BY分组同一学生的成绩-->
            <!--2. 扫描所有行！！返回结果。CONCAT(A, B)函数的作用是将表达式A和表达式B的计算结果​​拼接成一个字符串​。此处：‘未完成：11’字符串！！-->
            <!--4. AS：重新定义列名 -->
            SELECT CONCAT(
                CASE WHEN order_status = 0 THEN '未完成: '
                    ELSE '已完成: ' END,
                COUNT(*)
            ) AS result
            FROM orders
            WHERE order_status IN (0, 1)
            GROUP BY order_status;
            ```
        - explain plan
    - DCL数据控制语言：立即生效。
        - grant、revoke；
    - TCL事务控制语言：
        - START transaction/ BEGIN、SET transaction（事务隔离级别）
        - savepoint（设置保存点）、ROLLBACK TO savepoint、RELEASE savepoint
        - COMMIT、ROLLBACK

# 语句执行
- SQL的执行顺序是：
    - FROM
    - WHERE：过滤‘行’，不允许聚合函数使用
    - GROUP BY：分组
        ```sql
        <!-- select中只允许出现 group by中字段或者聚合函数。否则你想查name会产生一组中多个值，从而发生歧义。 -->
        SELECT dept, COUNT(*) AS employee_count
        FROM employees
        GROUP BY dept;
        ```
    - HAVING：AVG等聚合函数过滤‘分组数据’（和GROUP BY、WHERE前后顺序,必须结合group by使用。）
    - SELECT：真正获取数据列
    - ORDER BY：最后排序
    
- Java中操作数据库语句executeUpdate
    - 用PreparedStatement的executeUpdate()方法时，该方法的返回值是一个int类型，表示受影响的行数。
- 合并和过滤方法对比
    - ​​UNION ALL​​：垂直合并多个查询的结果集，不去重；
        - 要求每个 SELECT语句的列数、数据类型必须相同或兼容。
    - UNION：去重功能；
    - JOIN：水平连接多个表；
        - WHERE...OR：过滤行；
        ```sql
        SELECT e.name, d.department_name
        FROM employees e
        LEFT JOIN departments d ON e.department_id = d.id
        WHERE d.department_name = 'Sales' OR d.department_name IS NULL;
        ```

- MySQL函数
    - INSERT():INSERT('1234567', 1, 4, 'ab')的执行结果就是 ​'​ab567';
    - REPLACE():REPLACE('Hello World', 'World', 'MySQL');会返回 'Hello MySQL';
    - CONCAT():SELECT CONCAT('Hello', ' ', 'MySQL');会返回 'Hello MySQL';
