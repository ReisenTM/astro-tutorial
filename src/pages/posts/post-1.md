> 第**十**代博客

数据库:主从配置，读写分离
搜索：es

`git commit feat-[content]

## 配置初始化

使用yaml进行配置保存

```yaml
system:  
  ip:  
  port: 8080  
  env: dev # 一般用于gin的日志输出  
log:  
  app: blogx_server  
  dir: logs  

```

## 日志初始化

1. 为什么用日志库而不是标准日志
 - 功能更强大
 - 显示更清晰
2. 日志格式
 - logs/Date/App

## 连接数据库

==读写分离==
插件:<https://github.com/go-gorm/dbresolver>
能自动识别读写操作,进行操作分发

- settings.yaml

```yaml
db:  
  user: root  
  password: root  
  host: 127.0.0.1  
  port: 5432  
  database: gvb_db  
  debug: false  
  source: postgres  
# 数据库配置一样，模拟多个数据库读写分离  
db1:  
  user: root  
  password: root  
  host: 127.0.0.1  
  port: 5432  
  database: gvb_db  
  debug: false  
  source: postgres
```

**⚠️如果连接不上远程数据库**
查看🛜网络的代理设置，看是否设置了sock代理，可能连接被代理拦截了，增加白名单即可

## 路由初始化

将api从路由里剥离出来，避免耦合

## 根据ip获取地理位置

经常用于社交平台
通过ip地址去定位
在网上冲浪都是走公网ip
哪些是内网？

```
192.168.0.0
172.16-32
10.。。
127.0.0.1
```

ip2region: "github.com/lionsoul2014/ip2region/binding/golang/xdb"

- 离线数据库查询，效率高，精度低
或利用现有网站去发请求
- 精准度高，效率低

初始化ip2region

```go
func InitIPDB() {  
    var dbPath = "init/ip2region.xdb"  
    _searcher, err := xdb.NewWithFileOnly(dbPath)  
    if err != nil {  
       logrus.Fatalf("ip地址数据库加载失败: %s\n", err)  
       return  
    }  
    //不关闭因为后面还需要用  
    //defer searcher.Close()  
    searcher = _searcher  
}  
```

通过区间判断是否是内网

```go
func HasLocalIPAddr(ip string) bool {  
    return HasLocalIP(net.ParseIP(ip))  
}  
  
// HasLocalIP 通过ip判断内网  
func HasLocalIP(ip net.IP) bool {  
    if ip.IsLoopback() {  
       return true  
    }  
    ip4 := ip.To4()  
    if ip4 == nil {  
       return false  
    }  
    return ip4[0] == 10 ||  
       (ip4[0] == 172 && ip4[1] >= 16 && ip[4] <= 31) ||  
       (ip4[0] == 192 && ip4[1] == 168) ||  
       (ip4[0] == 169 && ip4[1] == 254)  
}
```

如果不是内网，再进行查表

```go
var searcher *xdb.Searcher  
const LOCFOMMAT = 5  
func GetIPLoc(ip string) (location string) {  
    //利用区间先快速判断是否是内网  
    if ipUtils.HasLocalIPAddr(ip) {  
       return "内网"  
    }  
    region, err := searcher.SearchByStr(ip)  
    if err != nil {  
       logrus.Warnf("错误的ip地址:[%s]", ip)  
       return "异常地址"  
    }  
    //处理addrList  
    _addrList := strings.Split(region, "|")  
    if len(_addrList) != LOCFOMMAT {  
       //出现概率目前极低  
       logrus.Warnf("异常的ip地址:[%s]", ip)  
       return "未知地址"  
    }  
 //...
 //处理格式
 //...
    return region  
}
```

## 表结构搭建

1. 优先核心表
2. 表的设计尽量保证后期表结构不变化
3. 要考虑**冗余字段**：==字段可以多但是不能少==

## 日志系统

- 登录日志
- 操作日志
 	- 获取请求体->可以放在请求中间件里

- 静态路由
 	- r.static(),路径映射URL

## JWT

jwt库
"github.com/dgrijalva/jwt-go"

1. **定义 Claims 结构**
自定义 Claims，包含用户的基本信息（如 UserID、Username、Role），并组合 jwt.StandardClaims 形成 MyClaims。

```go
type Claims struct {  
    UserID   uint   `json:"userID"`  
    Username string `json:"username"`  
    Role     uint8  `json:"role"`  
}  
  
type MyClaims struct {  
    Claims  
    jwt.StandardClaims  
}
```

1. **生成token**

```go
// GetToken 转换 token
func GetToken(claims Claims) (string, error) {  
    cla := MyClaims{  
       Claims: claims,  
       StandardClaims: jwt.StandardClaims{  
          ExpiresAt: time.Now().Add(time.Duration(global.Config.Jwt.Expire) * time.Hour).Unix(), // 过期时间  
          Issuer:    global.Config.Jwt.Issuer,                                                   // 签发人  
       },  
    }  
    //设置签名算法  
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, cla)  
    return token.SignedString([]byte(global.Config.Jwt.Secret)) // 进行签名生成对应的token  
}
```

3. **解析 Token**
通过 ParseToken(tokenString) 解析 JWT：

- 校验签名是否合法
- 判断是否过期、是否非法或无效
- 成功后返回自定义的 MyClaims

```go
func ParseToken(tokenString string) (*MyClaims, error) {  
    if tokenString == "" {  
       //如果未登录，直接返回  
       return nil, errors.New("请登录")  
    }  
    token, err := jwt.ParseWithClaims(tokenString, &MyClaims{}, func(token *jwt.Token) (interface{}, error) {  
       return []byte(global.Config.Jwt.Secret), nil  
    })  
    if err != nil {  
       //如果出错,判断出错类型  
       if strings.Contains(err.Error(), "token is expired") {  
          return nil, errors.New("token过期")  
       }  
       if strings.Contains(err.Error(), "signature is invalid") {  
          return nil, errors.New("token无效")  
       }  
       if strings.Contains(err.Error(), "token contains an invalid") {  
          return nil, errors.New("token非法")  
       }  
       return nil, err  
    }  
    //断言确定token有效  
    if claims, ok := token.Claims.(*MyClaims); ok && token.Valid {  
       return claims, nil  
    }  
    return nil, errors.New("invalid token")  
}
```

> 拓展：双token
>
> - 单token 过期时间长，安全程度低
> - 双token包括access_token用来鉴权，refresh_token用来获取新access_token，时间可以设置为较长

jwt缺点：

1. 不能主动失效
 Redis添加黑名单

## 图片管理

1. 图片上传的文件名重复问题
 - 直接存hash

## 七牛云

***TODO***

## 用户管理

### 图片验证码库:"github.com/mojocn/base64Captcha"

1. 存储器设置

```go
// 根据自己需求更改验证码存储上限和过期时间  
var result = base64Captcha.NewMemoryStore(10240, 3*time.Minute)  
//或者使用默认配置 10240,10min
var result = base64Captcha.DefaultMemStore
```

2. 生成器配置

```go
// digitConfig 生成图形化数字验证码配置  
func digitConfig() *base64Captcha.DriverDigit {  
    digitType := &base64Captcha.DriverDigit{  
       Height:   50,  
       Width:    100,  
       Length:   5,  
       MaxSkew:  0.45,  
       DotCount: 80,  
    }  
    return digitType  
}
```

3. 生成

```go
// CreateCode  
// @Result id 验证码id  
// @Result bse64s 图片base64编码  
// @Result err 错误  
func CreateCode() (string, string, string, error) {  
    var driver base64Captcha.Driver  
    //纯数字验证码  
    driver = digitConfig()  
    if driver == nil {  
       logrus.Errorf("图形化数字验证码配置失败")  
    }  
    // 创建验证码并传入创建的类型的配置，以及存储的对象  
    c := base64Captcha.NewCaptcha(driver, result)  
    id, b64s, answer, err := c.Generate()  
    return id, b64s, answer, err  
}
```

## 发邮件

```go
import (
    "log"
    "net/smtp"

    "github.com/jordan-wright/email"
)

func main() {
    e := email.NewEmail()
    //设置发送方的邮箱
    e.From = "dj <XXX@qq.com>"
    // 设置接收方的邮箱
    e.To = []string{"XXX@qq.com"}
    //设置抄送如果抄送多人逗号隔开
    e.Cc = []string{"XXX@qq.com",XXX@qq.com}
    //设置秘密抄送
    e.Bcc = []string{"XXX@qq.com"}
    //设置主题
    e.Subject = "这是主题"
    //设置文件发送的内容
    e.Text = []byte("www.topgoer.com是个不错的go语言中文文档")
    //设置服务器相关的配置
    err := e.Send("smtp.qq.com:25", smtp.PlainAuth("", "你的邮箱账号", "这块是你的授权码", "smtp.qq.com"))
    if err != nil {
        log.Fatal(err)
    }
}
```

setting.yaml配置

```yaml
email:  
  domain: ""  # 邮件服务器 
  port: 0   # 465 开ssl 587 没有开ssl
  sendEmail: ""   
  authCode: ""  
  sendNickname: ""  
  ssl: false  
  tls: false
```

- 用户密码一定不能明文存储在数据库里

## QQ登录

***TODO***

1. 随便做个页面：有qq登录就行
2. 审核

## 用户名+密码登录

## 命令行创建用户

包:"golang.org/x/crypto/ssh/terminal"
> 实现了密码的加密显示(" * ")

```go
terminal.ReadPassword(int(os.Stdin.Fd()))
```

## 邮箱绑定

## [PostgreSQL主从配置](obsidian://open?vault=Obsidian%20Vault&file=postgresql%E4%B8%BB%E4%BB%8E%E9%85%8D%E7%BD%AE")

## es配置

### **🔍 1.**  **强大的全文搜索能力**

- 支持模糊查询、分词、匹配度排序。
- 能处理拼写错误、同义词等复杂搜索需求。

### **⚡ 2.**  **高性能**

- 数据检索速度快，尤其适合处理大规模数据。
- 查询和写入都具备良好的延迟控制

#### docker-compose

```yaml
  es:
    image: "elasticsearch:7.12.0"
    restart: always
    privileged: true
    environment:
      discovery.type: single-node
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
    volumes:
      - ./es/data:/usr/share/elasticsearch/data
    networks:
      blogx_network:
        ipv4_address: 10.2.0.5
```

#### 修改data目录的权限[#](https://www.cnblogs.com/zydev/p/16039565.html#%E4%BF%AE%E6%94%B9data%E7%9B%AE%E5%BD%95%E7%9A%84%E6%9D%83%E9%99%90)

`chmod -R 0777 ./es/data`

[es v9笔记](obsidian://open?vault=Obsidian%20Vault&file=golang%20ES%20v9%20%E7%AC%94%E8%AE%B0)

## Mysql同步数据到es的方式

1. 同步双写
2. 异步双写

3. 数据抽取
4. 数据订阅
 1. canal( 选用)

## PG同步es

****TODO****

## 防Xss注入

## go Markdown 解析

md->html->text
rune?
**`rune`** 是一个内置的数据类型，用于表示 **Unicode 字符**（Unicode code point）。它的本质是 `int32` 的别名（占 4 个字节），用来处理 UTF-8 编码的字符

## 文章引入缓存

避免频繁访问数据库
引入缓存：浏览量，点赞数等
>如:点赞的时候，在缓存里面记录一个key,value，key 就是文章id，value 就是点赞数

查询文章的时候，从缓存里面查点赞数，响应的时候，实际点赞数=数据库中点赞数＋缓存中的点赞
数
在每天的0点进行数据同步

## 文章删除

1. ﻿﻿管理员可以删任意的文章，如果删的是用户的，应该给用户发一个系统消息
2. ﻿﻿﻿用户只能删自己发布的文章

>删文章如果是物理删除，就需要删除对应的关晚记录
文章点赞，文章收藏，文章置顶，文章评论，文章浏览

## 定时任务

库:Cron
用于在非高峰期同步缓存数据到数据库

## 全局通知

方案:

1. 管理员创建一条全局通知就给所有用户发一条系统消息(只适用于人少的情况)
2. 用户在系统消息界面主动查询

## WebSocket

websocket是[socket](https://so.csdn.net/so/search?q=socket&spm=1001.2101.3001.7020)连接和http协议的结合体，可以实现网页和服务端的长连接
> 不要在中间件加任何认证,认证通过查请求头去认证

```go
go get github.com/gorilla/websocket
```
<https://www.fengfengzhidao.com/article/htkS14sBEG4v2tWkYG4A#websocket>

建立连接的时机？
维护用户在线表

如果目的方在线，就建立ws连接
否则直接入库

## 文章查询

es,降级(db)

分数->相关度

## 获取服务器资源占用量

包：`github.com/shirou/gopsutil`
