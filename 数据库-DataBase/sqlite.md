# Sqlite

##### 1. 初始化数据库

```c++
int sqlite3_initialize(void);
int sqlite3_shutdown(void);
```

在使用SQlite 之前应该先调用`sqlite_initialize`函数将初始化一些必要数据结构的资源分配好。`sqlite3_shutdown`该函数用来释放由`sqlite_initialize`分配的资源。

如果直接使用`sqlite3_open`会自动初始化SQlite如果他还没被初始化的话。



##### 2. 打开数据库

```C++
int sqlite3_open(
  const char* filename,	 /*数据库的文件名(UTF8)*/
  sqlite3 **ppdb		/*数据库指针，传入指针的地址，用于给指针赋值, OUT: SQLite db handle*/
);

int sqlite3_open16(
  const void *filename, /*数据库的文件名(UTF16)*/
  sqlite3 **ppdb	    /*数据库指针，传入指针的地址，用于给指针赋值, OUT: SQLite db handle*/
);

int sqlite3_open_v2(
  const void *filename, /*数据库的文件名(UTF16)*/
  sqlite3 **ppdb	    /*数据库指针，传入指针的地址，用于给指针赋值, OUT: SQLite db handle*/
  int flags,
  const char *zVfs
);
```

`sqlite3_open` 函数假定SQlite3数据库文件名为UTF-8编码， `sqlite3_open_v2`是它的加强版。`sqlite_open16`假定SQlite3数据库文件名为UTF-16（Unicode宽字符）编码。

参数filename是要连接的SQlite3数据库文件名字符串。参数ppDb看起来有点复杂，它是一个指向指针的指针。

`sqlite3_open_v2`这个v2版本的函数强大就强大在它**可以对打开（连接）数据库的方式进行控制**，具体是通过它的参数**flags**来完成。`sqlite3_open_v2`函数只支持UTF-8编码的SQlite3数据库文件。

**flags：**

1.  `SQLITE_OPEN_READONLY` 则SQlite数据库文件会以只读方式打开，如果数据库文件不存在，则`sqlite3_open_v2` 函数执行失败，返回一个error
2. `SQLITE_OPEN_READWRITE` SQLite数据库文件会以可读可写的方式贷款，如果数据库文件本身备操作系统设置为保护模式，则以只读的方式打开；如果数据库文件不存在则`sqlite3_open_v2`函数执行失败返回error
3. `SQLITE_OPEN_READWRITE | SQLITE_OPEN_CREATE` 则SQLite数据库文件会以可读可写的方式打开，如果数据库文件不存在则新创建一个。这也是`sqlite3_open`和`sqlite3_open16`函数的默认行为。

**zVfs:**

​	参数zVfs允许客户应用程序命名一个虚拟文件系统（Virtual File System）模块，用来与数据库连接。**VFS**作为SQlite library和底层存储系统（如某个文件系统）之间的一个抽象层，通常客户应用程序可以简单的给该参数传递一个NULL指针，以使用默认的VFS模块。

##### 3. 关闭数据库

```c++
int sqlite3_close(sqlite3 *);	
```

在使用完SQlite数据库之后，需要调用`sqlite3_close`函数关闭数据库连接，释放数据结构所关联的内存，删除所有的临时数据项

如果在调用`sqlite3_close`函数关闭数据库之前，还有某些没有完成的（nonfinalized）SQL语句，那么sqlite3_close函数将会返回**SQLITE_BUSY**错误。

客户程序员需要finalize所有的预处理语句（prepared statement）之后再次调用`sqlite3_close`。

```objective-c
- (BOOL)close:(void *)_db {
    if (!_db) {
        return YES;
    }
    int  rc;	
    BOOL retry;
    BOOL triedFinalizingOpenStatements = NO;
    
    do {
        retry   = NO;
        rc      = sqlite3_close(_db);
      	// 数据库真正被使用，有未完成的sql语句
        if (SQLITE_BUSY == rc || SQLITE_LOCKED == rc) {
            if (!triedFinalizingOpenStatements) {
                triedFinalizingOpenStatements = YES;
                sqlite3_stmt *pStmt;
              	// 尝试关闭所有的预处理语句(prepared statement)
                while ((pStmt = sqlite3_next_stmt(_db, nil)) !=0) {
                    NSLog(@"Closing leaked statement");
                    sqlite3_finalize(pStmt);
                    retry = YES;
                }
            }
        }
        else if (SQLITE_OK != rc) {
            NSLog(@"error closing!: %d", rc);
        }
    }
    while (retry);
    
    _db = nil;
    return YES;
}
```

##### 5.执行SQL语句

SQLite数据库连接完成之后就可以执行SQL命令，操作SQL的流程

```c++
// 1. 为SQL创建一个声明(statement)
sqlite3_stmt *stmt = NULL;
sqlite3_prepare_v2(db, sql_str, sql_str_len, &stmt, NULL);

// 2. 
while(....) {
  // 绑定参数
  sqlite3_bind_xxx(stmt, param_idx, param_value...);
  // 
  while(sqlite3_step(stmt) == SQLITE_ROW) {
    col_val = sqlite3_column_xxx(stmt, col_index);
    ....
  }
  sqlite3_reset(stmt);
  sqlite3_clear_bindings(stmt);
}
// 销毁并释放声明 statement
sqlite3_finalize(stmt);
stmt = NULL;
```

1. `sqlite3_prepare_v2`：将一个SQL语句字符串转换为一条**prepared**语句，存储在`sqlite3_stmt`类型结构中。
2. `sqlite3_bind_xxx`: 给第一步生成的**prepared**语句绑定参数
3. `sqlite3_step` : 执行**prepared**语句，获取结果集中每一行数据，从每一行数据中调用`sqlite3_column_xxx`函数获取有用的列数据，直到结果集中所有的行都被处理完毕

**prepared**语句可以被重置(调用`sqlite3_reset`函数)，然后可以重新绑定参数重新执行。

`sqlite3_prepare_v2`函数代价昂贵，所以通常尽可能重用**prepared**语句。最后这条**prepared**语句确实不再使用时，调用`sqlite_finalize`函数释放所有的内容资源和`sqlite3_stmt`数据结构，删除**prepare**语句

**预处理**

```c++
int sqlite3_prepare(
  sqlite3 *db,  	 // 数据库指针(datebase handle)
  const char *zSql,  // SQL statement, UTF8 encode
  int nByte,         // Maximum length of zSql in bytes
  sqlite3_stmt **stmt,	// Stmt handle
  const char **pzTail	// Pointer to unused portion of zSql
);

int sqlite3_prepare_v2(
  sqlite3 *db,  	 // 数据库指针(datebase handle)
  const char *zSql,  // SQL statement, UTF8 encode
  int nByte,         // Maximum length of zSql in bytes
  sqlite3_stmt **stmt,	// Stmt handle
  const char **pzTail	// Pointer to unused portion of zSql
);

int sqlite3_prepare16(  
  sqlite3 *db,            /* Database handle */  
  const void *zSql,       /* SQL statement, UTF-16 encoded */  
  int nByte,              /* Maximum length of zSql in bytes. */  
  sqlite3_stmt **ppStmt,  /* OUT: Statement handle */  
  const void **pzTail     /* OUT: Pointer to unused portion of zSql */  
);  
  
int sqlite3_prepare16_v2(  
  sqlite3 *db,            /* Database handle */  
  const void *zSql,       /* SQL statement, UTF-16 encoded */  
  int nByte,              /* Maximum length of zSql in bytes. */  
  sqlite3_stmt **ppStmt,  /* OUT: Statement handle */  
  const void **pzTail     /* OUT: Pointer to unused portion of zSql */  
);  
```

上面的函数是将SQL字符串转换为**prepared**语句。

参数：

*db*是由`sqlite3_open`函数返回的指向数据库连接的指针

*zSql*是UTF-8或者UTF-16编码的SQL命令字符串

*nBytes*是zSql的字节长度，如果*nByte*为负值，则**prepare**函数会自动计算出*zSql*字节长度，要确保*zSql*传入的是以NULL结尾的字符串。如果SQL命令字符串中只包含一条SQL语句，那么它没有必要以“;”结尾。

*ppStmt*是一个指向指针的指针，用来传回一个指向新建的`sqlite3_stmt`结构体的指针，`sqlite3_stmt`结构体中保存有转换好的SQL语句。如果SQL命令字符串包含多条SQL语句，同时参数pzTail不为NULL，那么它将指向SQL命令字符串中的下一条SQL语句。

**Step**

```C++
int sqlite3_step(sqlite3_stmt*)	
```

`sqlite3_prepare`函数将SQL命令字符串解析并转为一系列命令字节码，这些字节码最终被传到Sqlite3的虚拟数据库引擎(VDBE: Virtual DataBase Engine)中执行，完成这项工作的是`sqlite3_Step`函数。

比如一个SELECT查询操作，`sqlite3_step`函数的每次调用都会返回结果集中的其中一行，直到再没有有效数据行了。每次调用`sqlite3_step`函数如果返回`SQLITE_ROW`，代表获得了有效数据行，可以通过`sqlite3_column`函数提取某列的值。如果调用`sqlite3_step`函数返回`SQLITE_DONE`，则代表**prepared	**语句已经执行到终点了，没有有效数据了。

很多命令第一次调用`sqlite3_step`函数就会返回`SQLITE_DONE`，因为这些SQL命令不会返回数据。对于INSERT，UPDATE，DELETE命令，会返回它们所修改的行号——一个单行单列的值。

**结果列**

```C++
int sqlite_column_count(sqlite3_stmt *pStmt);
```

返回结果集的列数

```c++
const char *sqlite3_column_name(sqlite3_stmt*, int N);
const char *sqlite3_column_name16(sqlite3_stmt*, int N);
```

返回结果集中指定列的列名，列的序号以0开始。比如一条SQL语句：SELECT pid AS person_id… 那么调用`sqlite3_column_name`函数返回结果集中第0列的列名就是person_id。返回的字符串指针将一直有效，直到再次调用`sqlite3_column_name`函数并再次读取该列的列名时失效。

```c++
int sqlite3_column_type(sqlite3_stmt*, int iCol);	
```

 该函数返回结果集中指定列的本地存储类型，如SQLITE_INTEGER，SQLITE_FLOAT，SQLITE_TEXT，SQLITE_BLOB，SQLITE_NULL。为了获取正确的类型，*该函数应该在任何试图提取数据的函数调用之前被调用。SQlite3数据库允许不同类型的数据存储在同一列中，所以对于不同行的相同索引的列调用该函数获取的列类型可能会不同*



**绑定值参数**

语句参数（statement parameters）是指插入到SQL命令字符串中的特殊字符，他们作为临时占位符当一条语句在**prepare**之后，尚未执行之前，可以给这些占位符绑定指定的值。

**参数符号**

语句参数一共有5种类型，它们跟随SQL命令字符串一起被传入到`sqlite3_prepare`函数

1. ?

   一个自动索引的匿名参数，如果一条语句含有多个“？”参数，则它们被隐式的赋予索引1，2，3...

   ```sql
   INSERT	INTO people (id, name)  VALUES(?, ?	)
   这里的两个 ？ 分别代表id的值和name的值。需要注意的是SQL命令字符串中的?语句参数的书写，不要带单引号，'?'只是一个单字符文本值，并不是一个语句参数。
   ```

   当这条SQL命令字符串**prepare**之后，就可以给这两个“?”语句参数绑定合适的值，之后调用step函数执行语句。

2. ?<index>

   具有显示数字索引的语句参数。?<index>与？相比主要优点是在一条SQL命令字符串中可以有多个具有相同索引的问号语句参数，如一条SQL命令包含多个"?1", 这就允许在同一条语句中，在多个语句参数所占据的位置绑定相同的值

   ```sql
   INSERT INTO people (pid, uid, name) VALUES (?1, ?1, ?2);	pid, uid 是同样值，都是第一个参数值
   ```

   ?<index> 的index值也不必连续

   ```sql
   INSERT INTO people (pid, uid, name) VALUES (?1, ?2, ?4);
   ```

3. :<name>

   ```sql
   INSERT INTO people (id, name) VALUES (:id, :name);   “:id”代表id的值，“:name” 代表name的值。
   ```

4. @<name> 用法与:<name>类似

5. $<name>

     这是用来支持Tcl变量的扩展语法，除非使用Tcl编程，否则推荐使用“:<name>”版本。

   ​	以上5种类型的语句参数，在使用的时候选择其中一种，并始终使用它。最好不要在一条语句中穿插使用多种形式的语句参数，这样会造成视觉混淆。**推荐使用“:<name>”版本，因为这种形式的语句参数看起来更直观。**

**绑定值**

​	在一条带参数语句**被prepare之后，step之前**，可以给其中的每一个参数绑定一个指定的值。如果一条语句已经调用**sqlite3_step**函数执行了，那就不能给这条语句中的参数绑定具体的值了，除非这条语句被重置。

​	一共有如下9个bind函数，所有这些函数的第1个参数，第2个参数和返回值都是相同的。第一个参数是指向`sqlite3_stmu`结构体的指针，第2个参数是要绑定的参数索引值，记住索引值是从1（而不是0）开始的。第3个参数是要赋值给参数的绑定值。第4个参数（如果有的话），代表第三个参数“绑定值”的字节长度。第5个参数（如果有的话），它是一个指向内存管理回调函数的指针。所有的这些bind函数，如果执行成功则返回`SQLITE_OK`，否则返回一个整形错误码。

```c++
int sqlite3_bind_blob(sqlite3_stmt*, int, const void*, int n, void(*)(void *))
```

绑定一个任意长度的BLOB类型的二进制数据。（BLOB：二进制大对象，相当于一个可以存储大量二进制数据的容器。）

```c++
int sqlite3_bind_dluble(sqlite3_stmt*, int, double);
```

绑定一个64位浮点值。

```c++
int sqlite3_bind_int(sqlite3_stmt*, int, int)
```

  绑定一个32位有符号整型值。

```c++
int sqlite3_bind_null(sqlite3_stmt*, int)
```

 绑定NULL。

```c++
int sqlite3_bind_text(sqlite3_stmt*, int, const char*, int n, void (*)(void *))
```

  绑定一个任意长度的UTF-8编码的文本值，第4个参数是**字节长度**，注意不是字符长度。如果给第4个参数传递**负值**，SQlite就会自动计算绑定值的字节长度（不包括NULL结尾符）。

```c++
int sqlite3_bind_zeroblob(sqlite3_stmt*, int, int n);  
```

 绑定一个任意长度的BLOB类型的二进制数据，它的每一个字节被置0。第3个参数是字节长度。这个函数的特殊用处是，创建一个大的BLOB对象，之后可以通过BLOB接口函数进行更新。

```c++
int sqlite3_bind_value(sqlite3_stmt*, int, const sqlite3_value*);
```

绑定`sqlite3_value`**结构体类型**的值，`sqlite3_value`结构体可以保存**任意格式**的数据。

对于text和blob类型的bind函数，绑定值传递的是一个buffer指针，通常这个buffer指针一定要保证有效，**直到该语句参数绑定了一个新值或者语句被finalize销毁**，对于这两类bind函数，第5个参数是对这个buffer的一个控制

如果第5个参数传递的是`NULL`或者`SQLITE_STATIC`常量，则**SQlite会假定这块Buffer是静态内存，或者客户端应用程序会小心管理和释放这块内存buffer，所以SQLite不管**

如果第5个参数传递的是`SQLITE_TRANSIENT`常量，则SQlite会在内部复制这块Buffer内容。这就允许客户端应用程序在调用完bind函数之后立即释放这块buffer（或者是一块栈上的buffer在离开作用域之后自动销毁）。SQlite会自动在合适的时机释放它内部复制的这块buffer。

对于第5个参数的最后一种选择是传递一个有效的“void mem_callback(void *ptr)”函数指针。当SQlite使用完这块buffer并打算释放它的时候，第5个参数传递的函数指针所指向的函数将会被调用。比如这块buffer是由`sqlite3_malloc`函数或者`sqlite3_realloc`函数分配的，则可以直接传递`sqlite3_free`函数指针给bind函数的第5个参数。如果是由其它系列的内存管理函数分配的内存，则应该传递其相应的内存释放函数。

针对bind函数使用的索引值，有以下几个参数

```c++
int sqlite3_bind_parameter_count(sqlite3_stmt*);  
```

  返回一个整数，指明一条语句中所使用的参数的最大索引值。

```c++
int sqlite3_bind_parameter_index( sqlite3_stmt *stmt, const char *name)
```

  返回一个命名参数（如：":pid"）的索引值。注意这第2个参数是UTF-8编码的，即使针对UTF-16编码的语句，第2个参数也要以UTF-8编码的字符串赋值。如果没有找到匹配名字的参数，该函数返回0

```c++
sqlite3_bind_int(stmt, sqlite3_bind_parameter_index(stmt, ":pid"), pid);
```

```c++
const char* sqlite3_bind_parameter_name( sqlite3_stmt *stmt, int pidx )
```

返回指定索引参数的文本名称，以UTF-8编码。

```c++
int sqlite3_clear_bindings( sqlite3_stmt *stmt )  
```

 如果想清空一条语句中所有参数所绑定的值，调用`sqlite3_clear_bindings`函数，该函数调用之后，语句中所有参数都绑定`NULL`值。该函数总是返回`SQLITE_OK`。

**如果想确保绑定到参数的值，不会引起内存泄露。最好在每次重置语句时，清空所有参数绑定。**

**安全性和性能**

 构造一条SQL命令字符串，并修改其中的某些值，除了使用上面的语句参数的方式，还有一种方法就是使用诸如c语言的字符串处理函数，如：

```c++
// c语言的字符串处理函数
snprintf(buf, buf_size,           
       "INSERT INTO people( id, name ) VALUES ( %d, '%s' );",           
       id_val, name_val); 	
//   假如为id_val, name_va作如下赋值：
id_val = 23;  
name_val = "Fred"; 
//  则得到的存储在buf中的SQL语句如下：  
INSERT INTO people( id, name ) VALUES ( 23, 'Fred');  
```

那么使用语句参数的方式，和使用字符串处理函数的方式相比，有什么好处呢？主要有以下三点：

  （1） 使用“语句参数”方式，具有更高的安全性，可以有效防止“SQL注入攻击”。 “SQL注入攻击”要想达到目的，就必须让`attack value`随着SQL命令字符串一起传送进SQL解析器。黑客如果在一条SQL命令字符串被送入到`sqlite3_prepare`函数之前，利用c字符串处理函数等途径将`attack` value注入其中，而在`sqlite3_prepare`函数之中进行解析（parse），就可以达到攻击目的。而使用“语句参数”方式，被传送到`sqlite3_prepare`函数的只是SQL命令字符串中的参数符号（如：“?”），而不是具体的值。在`sqlite3_prepare`函数执行之后，才会使用bind函数给参数符号绑定具体的值，这就可以避免attack value随着SQL命令字符串一起在`sqlite3_prepare`函数中被解析，从而有效躲避“SQL注入攻击”。

  （2）使用“语句参数”方式，可以更快的完成值替换。

  （3）使用“语句参数”方式，更节省内存。原因是使用如snprintf函数，需要一个SQL命令模板，一块足够大的输出缓存，而且字符串处理函数需要工作内存（working memory），除此之外对于整形，浮点型，特别是BLOBs，经常会占用更多的空间。

```c++
char *data = ""; /* default to empty string */  
sqlite3_stmt *stmt = NULL;  
int idx = -1;  
/* ... set "data" pointer ... */  
/* ... open database ... */  

rc = sqlite3_prepare_v2( db, "INSERT INTO tbl VALUES ( :str )", -1, &stmt, NULL );  
if ( rc != SQLITE_OK) exit( -1 );  

idx = sqlite3_bind_parameter_index( stmt, ":str" );  
sqlite3_bind_text( stmt, idx, data, -1, SQLITE_STATIC );  

rc = sqlite3_step( stmt );  
if (( rc != SQLITE_DONE )&&( rc != SQLITE_ROW )) exit ( -1 );  

sqlite3_finalize( stmt );  
/* ... close database ... */  
```

使用了参数绑定的方式，避免可能的“SQL注入攻击”。

**潜在的陷阱（Potential Pitfalls）**

```sql
INSERT INTO membership ( pid, gid, type ) VALUES ( :pid, :gid, :type );  
```

 这条SQL命令字符串在prepare之后，“:pid, :gid, :type”这三个参数全部绑定为NULL值。这条语句在执行之前，一定要给这三个参数绑定新的值。假如表membership的type这一列有默认值，那么有的程序员可能会有一个误解，假如上面这条语句在step执行时，参数“:type”绑定的值为NULL，那么最终插入到表membership的列type中的值，应该是该列的默认值。这种假设是错误的，实际插入的就是NULL，而不是该列的默认值。假如type列想插入默认值，正确的写法如下：

```sql
INSERT INTO membership ( pid, gid ) VALUES ( :pid, :gid );  	
```

另一种容易引起误用的情况是与NULL值的比较。

```sql
SELECT * FROM employee WHERE manager = :manager;  
```

这条语句看起来可以很好的工作，但当参数“:manager”绑定NULL值的时候，这个查询操作将不会检索到任何数据，即使表中存在manager为NULL的行。如果需要manager列与NULL值进行比较，正确的写法如下：

```sql
SELECT * FROM employee WHERE manager IS :manager;
```

[原文](http://blog.csdn.net/northcan/article/details/7235519)







