---
title: TiDB 大整数最佳实践
summary: 介绍大整数在 TiDB 中的应用
---

# 大整数最佳实践

## 大整数

现在大部分程序语言直接支持 64 bit 的内置类型，即直接支持相关运算。而 `大整数` 是超过 64 bit 的整数，一般是需要自己来实现。 

有些客户也有大整数的强需求，这里根据几个成功案例，总结大整数的解决方案。

## 大整数 TiDB 解决方案

TiDB 没有直接支持大整数的存储与计算（默认支持的最大整数是 `64bit`），针对 大整数 比较通用的解决方案是： 
 
- 用 `binary(M)` 来存储大整数 —— `M` 表示大整数的字节数（比如，`U256` 用 `binary(32)` 来表示）。
- **必须** 高位补 0，保证 M 个字节全部填满。
- **必须** 是 `big-endian` 字节序列—— 这个由应用保证，生成的字节序列是 `big-endian` 的。  
具体实现细节、原理请看下文。  

## 大整数实现原理  

### 1. 存储

TiDB 默认支持的最大整数类型是 64bit，所以跟大多数程序语言一样，是默认不支持大整数的。  
但是，TiDB 是数据库，如果仅仅存储这类大整数数据，还是没问题的。  
在 TiDB 用 `binary(M)` 来存储大整数。其中，M 是根据 大整数 `bit 数` 除以 8 得到。  

> 一般情况下，大整数 `U/S + bit`位数一起来表示类型。其中，`U`表示无符号整数；`S`表示有符号整数。所以，U256 即表示 无符号的 256bit 的整数。当然，还可以有 有符号 128bit 整数 -\> S128。当然，计算机直接支持的，u64 表示 64bit 的无符号整数。  

> `binary(M)` ，`M` 表示字节数。所以，如果用 `binary(M)` 来表示 `U256`，那么就需要 32字节`(32=256/8)` =\> `binary(32)` 来存储 `U256`/`S256`。

### 2. 字节序(Byte Order)

关于字节序，网上资料、说明有许多，大家可以自行去了解。  

简单来说，0x03020100 需要 4字节存储，假设从位置 p 开始  

```C++
// 存储中存放示例
// big-enian 
p[0] = 0x03; 
p[1] = 0x02;
p[2] = 0x01;
p[3] = 0x00; // 大(高位)存储 放在了 小数据(低位数据)。
// little-endian
p[0] = 0x00; 
p[1] = 0x01;
p[2] = 0x02;
p[3] = 0x03;  // 大(高位)存储 放在了 大数据(高位数据)。
```

所以，很明显的，排序还跟 ByteOrder 相关。一般系统的默认 ByteOrder ，指的是系统内置类型整数的存储到 byte 中的顺序。而大整数，是我们自己实现的。  

而系统默认排序算法，一般都是从 bytes 的低字节开始到最后一个字节，一个个遍历比较。  
> binary 本质上来说也是 bytes，所以排序规则也是 从 `低字节(byte[0])` 到 `高字节(byte[len-1])` 一个个字节进行比较。
> 
> 我们知道，整数中，高位数据肯定比低位数据大（比如 `百位的 1` 比 `十位 的任何数字`都大）。所以正常的整数比较，都是高位数据跟高位数据比较 —— 这就要求 binary 存储数据，`高位数据` 存储在 `低位存储`，从上面示例可以知道，这是 `big-endian`!!!

### 3. 排序

我们已经知道，binary 必须要以 big-endian 存储。但是，这里还有一个注意点：不会比较长度。具体看如下例子：  
> A = 0x0203  
> B = 0x010203  
> 如果作为整数比较，那么 `0x0203 < 0x010203` 得到 `A < B`  
> 但是，如果 binary 比较，那么会先比较最高位字节即 `0x02 > 0x01` ，所以 `A > B`  
> 附注：如果最高位字节相等，那么比较下一个 字节；如此反复；  

所以，我们期望的是用 binary 比较 有 正数比较 相同的效果。其实，这里面最重要的点是没比较长度——既然它不比较，那么我们就弄成长度相等（高位补0），那就不需要比较长度了！！！  

其实，我们一般的程序语言内置类型（比如 u64），本质上来说就是 高位补0 了，所以即等长比较大小的。

> OK，用了定长数据，继续上面的例子，来看看效果：  
> 我们假设都是 4字节整数（u32)  
> A = `0x0000 0203`  
> B = `0x0001 0203`  
> 所以根据刚刚算法，`0x00 == 0x00` , 然后 `0x00 < 0x01` ,得到  `A < B` ，符合预期！！！。

### 4. 小结

所以，这里 TiDB 来处理大整数，两个基本的问题都解决了：

- 存储: binary(M) 来存储，M 可以根据大整数 bit数 来计算 。
- 字节序: 必须要以 big-endian 生成的 bytes ，来存储到 binary 中。
    - 注意，这个也是 应用来保证的。只有以这种方式存储，才能得到满足预期的结果
- 排序: 通过补0 来保证存储的 binary 长度一致，所以 binary 默认排序可以满足。
    - 注意，这里的补0不是字符串 `'0'(0x30)` ，而是数值`0(0x00)`; 而之前的示例都是用 hex 表示，即整数数据，而非字符串 -- **切记切记**。
    - 有了排序，存储的时候就可以按序存储，查询的时候，也可以高效的利用索引，而且查询的数据也会符合预期。
- 展示: 一般情况下，binary 一般是用 hex 格式展示
    - 如果是 非打印字符，那么一般默认用 hex 展示。
        - 我们这里的二进制数据，所以是没法打印的，所以只能用 hex
- 其它: 其它的种种 大整数功能，请应用程序来实现
    - 其它功能，包括但不限于： 基本的整数运算，安全(溢出)等等。

## 大整数 在应用程序实现

一般情况下，高级程序语言本身库来支持 大整数，而且支持一整套运算。  

> 比如， Golang 有 `big.Int` （系统库）；C/C++ 有 `gmp` ； Java 有 `BigInteger` 等。

- 基本功能: 应用端都是没问题的。
- 安全: 溢出等安全问题，一般 整数运算 类似，要注意。
- 存储: 要注意在转换的时候，要补0 来达到 特定字节数的 bytes。
- 字节序: 一定要是 big-endian （不过一般默认都是该字节序）。

## 大整数实战篇

1. 基本需求  
    业务侧需要存储`U256`大整数；如果超出，则截断.
2. 数据库侧

    ```sql
    -- 表结构设计
    CREATE TABLE IF NOT EXISTS myu256 (
        id int PRIMARY KEY AUTO_INCREMENT,
        idx int,
        raw BINARY(32),     -- 大整数实际存储的 bytes (没有高位补0).
        ud256 BINARY(32),   -- 补 0 之后的 bytes.
        rawhex varchar(64), -- raw 的 hex 数据(没有高位补 0 的 hex).
        hex varchar(64)     -- ud256 的 hex 数据.
    );
    ```

3. 业务侧  

    > 用 golang 做示例，本质上对 big.Int 进行封装，实现 U256 的基本功能 —— 可以通过 big.Int 来生成 `U256`；`U256` 整数生成 bytes（高位补 0 )，这样可以存储到数据库中的 BINARY(32)。

    ```golang
    // Package math provides integer math utilities.
    package bigint

    import (
        "math/big"
    )

    // Various big integer limit values.
    var (
        tt255     = BigPow(2, 255)
        tt256     = BigPow(2, 256)
        tt256m1   = new(big.Int).Sub(tt256, big.NewInt(1))
        tt63      = BigPow(2, 63)
        tt63m1    = new(big.Int).Sub(tt63, big.NewInt(1))
        tt64      = BigPow(2, 64)
        tt64m1    = new(big.Int).Sub(tt64, big.NewInt(1))
        MaxBig256 = new(big.Int).Set(tt256m1)
        MaxBig63  = new(big.Int).Sub(tt63, big.NewInt(1))
    )

    const (
        // number of bits in a big.Word
        wordBits = 32 << (uint64(^big.Word(0)) >> 63)
        // number of bytes in a big.Word
        wordBytes = wordBits / 8
    )

    // BigPow returns a ** b as a big integer.
    func BigPow(a, b int64) *big.Int {
        r := big.NewInt(a)
        return r.Exp(r, big.NewInt(b), nil)
    }

    // PaddedBigBytes encodes a big integer as a big-endian byte slice. The length
    // of the slice is at least n bytes.
    func PaddedBigBytes(bigint *big.Int, n int) []byte {
        if bigint.BitLen()/8 >= n {
            return bigint.Bytes()
        }
        ret := make([]byte, n)
        ReadBits(bigint, ret)
        return ret
    }

    // ReadBits encodes the absolute value of bigint as big-endian bytes. Callers must ensure
    // that buf has enough space. If buf is too short the result will be incomplete.
    func ReadBits(bigint *big.Int, buf []byte) {
        i := len(buf)
        for _, d := range bigint.Bits() {
            for j := 0; j < wordBytes && i > 0; j++ {
                i--
                buf[i] = byte(d)
                d >>= 8
            }
        }
    }

    // U256 encodes as a 256 bit two's complement number. This operation is destructive.
    func U256(x *big.Int) *big.Int {
        return x.And(x, tt256m1)
    }

    // U256Bytes converts a big Int into a 256bit EVM number.
    // This operation is destructive.
    func U256Bytes(n *big.Int) []byte {
        return PaddedBigBytes(U256(n), 32)
    }
    ```
    
   > U256 相关测试代码  

   > 1. TestU256 测试了 U256 的功能：基本的计算，溢出的时候截断等行为
   > 2. TestSortBytes 测试了 golang 本身的 bytes 排序规则
   > 3. TestOrder 测试了 U256 的 补 0 之后 bytes 的排序规则
   > 4. TestMysqlSort 测试了 U256 存储到 TiDB 之后的排序规则

    ```golang
    package bigint

    import (
        "bytes"
        "database/sql"
        "fmt"
        "math"
        "math/big"
        "sort"
        "testing"

        _ "github.com/go-sql-driver/mysql"
        "github.com/stretchr/testify/assert"
    )

    var (
        big1 = big.NewInt(1)
    )

    type TType struct {
        src    *big.Int
        expect int
    }

    // U256Bytes 只处理 256bit 的数据；超出部分，直接过滤；
    //      这里测试了 256bit 的最大值；然后 +1 即溢出转为0； +2 溢出之后的 1.
    //      原先值 v0,转换为 bytes 之后，再转化回来值 为 v1； expect 是 v0.Cmp(v1)
    //      注意，这里不测试负数.
    func TestU256(t *testing.T) {
        if math.MaxUint64 != tt64m1.Uint64() {
            t.Fatalf("math.MaxUint64 != tt64m1.Uint64()")
        }
        assert.Equal(t, int64(math.MaxInt64), tt63m1.Int64())
        var inputs = []TType{
            {big.NewInt(1), 0},
            {tt63m1, 0},
            {tt64m1, 0},
            {big.NewInt(0), 0},
            {MaxBig256, 0}, // max
            {big.NewInt(0).Add(big.NewInt(1), MaxBig256), 1}, // max + 1 , 被截断.
            {big.NewInt(0).Add(big.NewInt(2), MaxBig256), 1}, // max + 2 , 被截断.
        }
        tmp := big.NewInt(0)
        for idx, tt := range inputs {
            tmp.SetBytes(U256Bytes(big.NewInt(0).Set(tt.src)))
            r := tt.src.Cmp(tmp)
            if r != tt.expect {
                t.Errorf("idx=%d:\n"+
                    "input    =0x%x;\n"+
                    "  tmp    =0x%x;\n"+
                    "U256Bytes=0x%x;\n"+
                    "       r = %d.\n",
                    idx, tt.src.Bytes(), tmp.Bytes(), U256Bytes(tt.src), r)
            }
        }
    }

    type u256Array [][]byte

    func (u u256Array) Len() int {
        return len(u)
    }
    func (u u256Array) Less(i, j int) bool {
        return bytes.Compare(u[i], u[j]) < 0
    }
    func (u u256Array) Swap(i, j int) {
        u[i], u[j] = u[j], u[i]
    }

    var i64s = []int64{0x1, 0x21, 0x3, 0x111}

    // 用map，是为了 无序；
    var inputs = map[int]*big.Int{
        3: tt63m1,
        0: big.NewInt(0),
        5: big.NewInt(0).Add(tt64m1, big1),
        1: big.NewInt(1),
        4: tt64m1,
        2: big.NewInt(0xff),
    }

    // 主要是为了测试排序：转为 bytes 之后的排序是否符合预期。
    func TestOrder(t *testing.T) {

        // 这里 bytes 排序;
        var u256s u256Array
        for _, v := range inputs {
            tt := U256Bytes(v)
            u256s = append(u256s, tt)
            t.Logf("%x", tt)
        }
        sort.Sort(u256s)
        assert.Equal(t, len(u256s), len(inputs))
        var tmp = big.NewInt(0)
        for k, v := range inputs {
            tmp.SetBytes(u256s[k])
            if v.Cmp(tmp) != 0 {
                t.Errorf("idx=%d:\ninput    =0x%x;\nu256s[%d]=0x%x;\n",
                    k, v.Bytes(), k, u256s[k])
            }
        }
    }

    type MysqlU256 struct {
        idx    int
        row    []byte
        u256   []byte
        rawhex string
        hex    string
    }

    func newDefaultCleanTestDB(t *testing.T) *sql.DB {
        dsn := fmt.Sprintf("root:@tcp(127.0.0.1:4000)/test?charset=utf8&parseTime=True")
        dbo, err := sql.Open("mysql", dsn)
        if err != nil {
            t.Fatalf("open sql err:%s", err.Error())
        }
        // create table;
        _, err = dbo.Exec("create table if not exists myu256 (id int auto_increment primary key,idx int,raw BINARY(32),ud256 BINARY(32), rawhex varchar(64),hex varchar(64))")
        if err != nil {
            t.Fatalf("open sql err:%s", err.Error())
        }
        dbo.Exec("truncate table myu256")
        return dbo
    }

    // create table myu256(id int auto_increment primary key,idx int,raw BINARY(32),ud256 BINARY(32), rawhex varchar(64),hex varchar(64));

    func insertDB(t *testing.T, idx int, v *big.Int, db *sql.DB) {
        mu := MysqlU256{
            idx:    idx,
            row:    v.Bytes(),                       // 直接 bytes
            u256:   U256Bytes(v),                    // 256bit 的 bytes
            rawhex: fmt.Sprintf("%x", v.Bytes()),    // 直接 bytes 转 hex
            hex:    fmt.Sprintf("%x", U256Bytes(v)), // 256bit bytes 转 hex
        }
        // 占位符插入
        sql := "insert into myu256(idx,raw,ud256,rawhex,hex) values(?,?,?,?,?)"
        _, err := db.Exec(sql, &mu.idx, &mu.row, &mu.u256, &mu.rawhex, &mu.hex)
        if err != nil {
            t.Fatalf("exe(0x%s) err:%s", mu.hex, err.Error())
        }
    }

    func mysqlSortBy(t *testing.T, orderField string, db *sql.DB) {
        query := fmt.Sprintf(`select idx,%s from myu256 order by %s`, orderField, orderField)
        rows, err := db.Query(query)
        if err != nil {
            t.Fatalf("query(%s) err:%s", orderField, err.Error())
        }
        idx := 0
        var str string
        prev := 0
        first := true
        ok := true
        // var bs strings.Builder
        for rows.Next() {
            err = rows.Scan(&idx, &str)
            if err != nil {
                rows.Close()
                t.Fatalf("scan(%s) err:%s", orderField, err.Error())
            }
            // bs.WriteString(fmt.Sprintf("\t(%d,%s)\n", idx, str))
            if first {
                prev = idx
                first = false
            } else {
                if prev+1 != idx {
                    ok = false
                    // break
                } else {
                    prev = idx
                }
            }
        }
        rows.Close()

        if ok {
            t.Logf("order by '%s' ok ", orderField)
        } else {
            t.Logf("wrong order( by '%s' ), sql = '%s';", orderField, query)
        }
        // t.Logf("\t%s:\n%s", query, bs.String())
    }

    // 这里设计了一个表，raw 表示原始的 bytes(可能不足256 bit); ud256 表示经过 U256Bytes 的 bytes(256bit); rawhex 表示 转为 hex 的数据.  hex 表示 256bit 转为 hex.
    func TestMysqlSort(t *testing.T) {

        // to structs;
        db := newDefaultCleanTestDB(t)
        inputs := []*big.Int{
            big.NewInt(0),
            big.NewInt(0x1),
            big.NewInt(0x3),
            big.NewInt(0x21),
            big.NewInt(0xff),
            big.NewInt(0x1111),
            tt63m1,
            big.NewInt(0).Add(tt64m1, big1),
            big.NewInt(0).Add(tt64m1, big.NewInt(2)),
        }
        // 插入数据库;
        for id, v := range inputs {
            insertDB(t, id, v, db)
        }
        // 排序查询;
        for _, f := range []string{"raw", "ud256", "rawhex", "hex"} {
            mysqlSortBy(t, f, db)
        }
    }

    // 直接排序，是不符合预期的；
    func TestSortBytes(t *testing.T) {
        // var i64s = []int64{0x1, 0x21, 0x3, 0x111}
        var iarrs u256Array // sort = [01 0111 03 21 ]
        var ihexs []string  // sort = [01 0111 03 21]
        var tmp big.Int

        for _, i := range i64s {
            tmp.SetInt64(i)
            iarrs = append(iarrs, tmp.Bytes())
            ihexs = append(ihexs, fmt.Sprintf("%x", tmp.Bytes()))
        }
        sort.Sort(iarrs)
        sort.Strings(ihexs)
        var str string
        for _, v := range iarrs {
            str += fmt.Sprintf("%x ", v)
        }
        t.Log("iarrs=")
        t.Logf("[%s]", str)
        t.Log("ihex=")
        t.Log(ihexs)
        assert.Equal(t, len(iarrs), len(ihexs))
        for i := 0; i < len(iarrs); i++ {
            str := fmt.Sprintf("%x", iarrs[i])
            assert.Equal(t, ihexs[i], str)
        }
    }

    ```