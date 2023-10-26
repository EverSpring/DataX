# oracle增量同步使用
1. jar包替换
打包后替换`plugin-rdbms-util-0.0.1-SNAPSHOT.jar`和`oraclewriter-0.0.1-SNAPSHOT.jar`
2. 配置同步json ，添加`writeMode`参数，内容为`update (索引)`  
`update`括号中的根据什么字段更新，一般是索引字段。执行merge into的时候会自动拼接`column`的字段，这些字段也是要更新的字段
3. 配置样例如下  
表table_t中`LOGNO`列为主键
```json
{
  "job": {
    "setting": {
      "speed": {
        "channel": 3,
        "byte": 1048576
      },
      "errorLimit": {
        "record": 0,
        "percentage": 0.02
      }
    },
    "content": [
      {
        "reader": {
          "name": "oraclereader",
          "parameter": {
            "username": "xx",
            "password": "xx",
            "splitPk": "",
            "connection": [
              {
                "querySql": [
                  "select LOGNO, TRANCOM, TRANNO from table_test where LOGNO=3236817"
                ],
                "jdbcUrl": [
                  "jdbc:oracle:thin:@//127.0.0.1:1521/lis"
                ]
              }
            ]
          }
        },
        "writer": {
          "name": "oraclewriter",
          "parameter": {
            "username": "pp",
            "password": "pppp",
            "column": [
              "LOGNO",
              "TRANCOM",
              "TRANNO"
            ],
            "connection": [
              {
                "table": [
                  "table_test"
                ],
                "jdbcUrl": "jdbc:oracle:thin:@//10.2.0.45:1521/prip"
              }
            ],
            "writeMode": "update (LOGNO)"
          }
        }
      }
    ]
  }
}
```
以上配置会在同步时是以`merge into`执行
```sql
MERGE INTO table_test A
    USING (SELECT ? AS LOGNO FROM DUAL) TMP
    ON (TMP.LOGNO = A.LOGNO)
WHEN MATCHED THEN
    UPDATE
        SET TRANCOM       = ?,
            TRANNO        = ?,
            OPERATOR      = ?
WHEN NOT MATCHED THEN
    INSERT
        (LOGNO,
        TRANCOM,
        TRANNO)
    VALUES
        (?,
        ?,
        ?)
```
4. 常见错误
* 问题1：索引中丢失  IN 或 OUT 参数:: xxxx
在我遇到的场景是，我的json是由datax-web生成的，所在`column`生成的字段带有引号,比如：
```sql
"\"LOGNO\"",
"\"TRANCOM\"",
"\"TRANNO\""
```
* 问题2：java.lang.IndexOutOfBoundsException: Index: xxx, Size: xxx
这可能是`writeMode`的用法错误。比如
```
writeMode: update(TRANCOM,TRANNO)
```
update的字段因为是作为更新条件的列，而不是更新列
