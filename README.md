# verysimple

verysimple， 实际上 谐音来自 V2ray Simple (显然只适用于汉语母语者), 

verysimple项目大大简化了 转发机制，能提高运行速度。本项目 转发流量时，关键代码直接放在main.go里！非常直白易懂

只有项目名称是v2ray_simple，其它所有场合全 使用 verysimple 这个名称，可简称 "vs"。

规定，编译出的文件名必须以 verysimple 开头.

## 特点

实现了vless协议（v0，v1）和vlesss（即vless+tcp+tls），入口使用socks5协议

在本项目里 制定 并实现了 vless v1标准，添加了非mux的fullcone；

本项目 发明了独特的非魔改tls包的 双向splice

v0协议是直接兼容现有v2ray/xray的，比如可以客户端用任何现有支持vless的客户端，服务端使用verysimple

经过实际测速，就算不使用lazy encrypt等任何附加技术，verysimple作为服务端还是要比 v2ray做服务端要快。反过来也是成立的。

### 关于vless v1

这里的v1是我自己制定的，总是要摸着石头过河嘛。标准的讨论详见 [vless_v1](vless_v1.md)

在客户端的 配置url中，添加 `?version=1` 即可生效。

总之，强制tls，简单修订了一下协议格式，然后重点完善了fullcone。

我 实现了 一种独创的 非mux型“隔离信道”方法的 udp over tcp 的fullcone

测试 fullcone 的话，由于目前 verysimple 客户端只支持socks5入口，可以考虑先用v2ray + Netch或者透明代理 等方法监听本地网卡的所有请求，发送到 verysimple 客户端的socks5端口，然后 verysimple 客户端 再用 vless v1 发送到 v2simple vless v1 + direct 的服务端。



### 关于udp

本项目 vless 和 socks5 均支持 udp

最新的代码已经完整支持vless v0

后来我还自己实现了vless v1，自然也是支持udp的，也支持fullcone。v1还处于测试阶段

### tls lazy encrypt (splice) 

在最新代码里，还实现了 双向 tls lazy encrypt, 即另一种 xtls的 splice的实现，底层也是会调用splice，本包为了加以区分，就把这种方式叫做 tls lazy encrypt。

tls lazy encrypt 特性 运行时可以用 -lazy 参数打开（服务端客户端都要打开），然后可以用 -pdd 参数 打印 tls 探测输出

因为是双向的，而xtls的splice是单向，所以 理论上 tls lazy encrypt 比xtls 还快，应该是正好快一倍？不懂。反正我是读写都是用的splice。

而且这种技术不通过魔改tls包实现，而是在tls的外部实现，不会有我讲的xtls的233漏洞，而且以后可以与utls配合 进行模拟指纹。

关于 splice，还可以参考我的文章 https://github.com/hahahrfool/xray_splice-

该特性不完全稳定，可能会导致一些网页访问有时出现异常,有时出现bad mac alert;刷新页面可以解决

不是速度慢，是因为 目前的tls过滤方式有点问题, 对close_alert等情况没处理好。而且使用不同的浏览器，现象也会不同，似乎对safari支持好一些， chrome就差一些

在我的最新代码里，采用了独特的技术，已经规避了大部分不稳定性。总之比较适合看视频，毕竟双向splice，不是白给的！

经过我后来的思考，发现似乎xtls的splice之所以是单向的，就是因为它在Write时需要过滤掉一些 alert的情况，否则容易被探测；

不过根据 [a report by gfwrev](https://twitter.com/gfwrev/status/1327670741597179906), 对拷直连 还是会有很多问题，很难解决

所以既然问题无法解决，不如直接应用双向splice，也不用过滤任何alert问题。破罐子破摔。

总之这种splice东西只适用于玩一玩，xtls以及所有类似的 对拷直连的 技术都是不可靠的。我只是放这里练一下手。大家玩一玩就行。

我只是在内网自己试试玩一玩，从来不会真正用于安全性要求高的用途。

关于splice的一个现有“降速”问题也要看看，（linux 的 forward配置问题），我们这里也是会存在的 https://github.com/XTLS/Xray-core/discussions/59

**注意，因为技术实现不同，该功能不兼容xtls。**, 因为为了能够在tls包外进行过滤，我们需要做很多工作，所以技术实现与xtls是不一样的。

#### 总结 tls lazy encrypt 技术优点

解决了xtls以下痛点

1. 233 漏洞
2. 只有单向splice
3. 无法与fullcone配合
4. 无法与utls配合

原因：

1. 我不使用循环进行tls过滤，而且不魔改tls包
2. 我直接开启了双向splice；xtls只能优化客户端性能，我们两端都会优化;一般而言大部分服务器都是linux的，所以这样就大大提升了所有连接的性能.
3. 因为我的vless v1的fullcone是非mux的，分离信道，所以说是可以应用splice的（以后会添加支持，可能需要加一些代码，有待考察）
4. 因为我不魔改tls包，所以说可以套任何tls包的，比如utls

而且alert根本不需要过滤，因为反正xtls本身过滤了还是有两个issue存在，是吧。

而且后面可以考虑，如果底层是使用的tls1.2，那么我们上层也可以用 tls1.2来握手。这个是可以做到的，因为底层的判断在客户端握手刚发生时就可以做到，而此时我们先判断，然后再发起对 服务端的连接，即可。
也有一种可能是，客户端的申请是带tls1.3的，但是目标服务器却返回的是tls1.2，这也是有可能的，比如目标服务器比较老，或者特意关闭了tls1.3功能；此时我们可以考虑研发新技术来绕过，也要放到vless v1技术栈里。参见 https://github.com/hahahrfool/v2ray_simple/discussions/2

### ws/grpc

以后会添加ws/grpc的支持。并且对于ws/grpc，我设计的vless v1协议会针对它们 有专门的udp优化。
## 安装方式：

```go
git clone https://github.com/hahahrfool/v2ray_simple
cd v2ray_simple && go build
cp client.example.json client.json
cp server.example.json server.json
```

详细优化的编译参数请参考Makefile文件

如果你是直接下载的可执行文件，则不需要 go build了，直接复制example.json文件即可

### 关于内嵌geoip 文件

默认的Makefile或者直接 go build 是不开启内嵌功能的，需要加载外部mmdb文件，就是说你要自己去下载mmdb文件，

可以从 https://github.com/P3TERX/GeoLite.mmdb 项目，https://github.com/Loyalsoldier/geoip 项目， 或者类似项目 进行下载

加载的外部文件 必须使用原始 mmdb格式。

若要内嵌编译，要用 `tar -czf GeoLite2-Country.mmdb.tgz GeoLite2-Country.mmdb` 来打包一下，将生成的tgz文件放到 netLayer文件夹中，然后再编译 ，用 `go build -tags embed_geoip` 编译

内嵌编译 所使用的 文件名 必须是 GeoLite2-Country.mmdb.tgz


因为为了减小文件体积，所以才内嵌的gzip格式，而不是直接内嵌原始数据

## 使用方式

```sh
#客户端
v2ray_simple -c client.json

#服务端
v2ray_simple -c server.json
```

如果你不是放在path里的，则要 `./v2ray_simple`, 前面要加一个点和一个斜杠。windows没这个要求。

关于 vlesss 的配置，查看 server.example.json和 client.example.json就知道了，很简单的。

目前配置文件一共就4行，其中两行还是花括号，这要是还要我解释我就打你屁股

## 验证方式

对于功能的golang test，请使用 `go test ./...` 命令。如果要详细的打印出test的过程，可以添加 -v 参数


## 开发标准以及理念

文档尽量多，代码尽量少
### 文档

文档、注释尽量详细，且尽量完全使用中文，尽量符合golang的各种推荐标准。

根据golang的标准，注释就是文档本身（godoc的原理），所以一定要多写注释。不要以为解释重复了就不要写，因为要生成godoc文档，在 pkg.go.dev 上 给用户看的时候它们首先看到的是注释内容，而不是代码内容

本项目所生成的文档在 https://pkg.go.dev/github.com/hahahrfool/v2ray_simple

再次重复，文档越多越好，尽量降低开发者入门的门槛。

### 代码

代码的理念就是极简！这也是本项目名字由来！

根据 奥卡姆剃刀原理，不要搞一大堆复杂机制，最简单的能实现的代码就是最好的代码。


## 本项目所使用的开源协议

MIT协议，即你用的时候也要附带一个MIT文件，然后我不承担任何后果。

同时附带要求满足[此文件](No_PR_For_V2Ray_XRAY.md)中提到的要求

## 历史

启发自我fork的v2simple，不过原作者的架构还是有点欠缺，我就直接完全重构了，完全使用我自己的代码。

这样也杜绝了 原作者跑路 导致的 一些不懂法律的人对于开源许可的 质疑。

实际上是毫无问题的，关键是他们太谨慎。无所谓，现在我完全自己写，没话说了吧—；

我fork也是尊重原作者，既然你们这么谨慎，正好推动了我的重构计划，推动了历史发展
## 额外说明 以及 开发计划

verysimple 是一个很简单的项目，覆盖协议也没有v2ray全，比如socks协议只能用于客户端入口，没法用于出口。

本项目的目的类似于一种 proof of concept. 方便理解，也因为极简所以能比官方v2ray快一些。

也因为是poc，所以我有时会尝试向 verysimple 中添加一些我设计的新功能。目前正在计划的有

1. 完善并实现 vless v1协议
2. 什么时候搞一个 verysimple_c 项目，用c语言照着写一遍; 也就是说，就算本verysimple没有任何技术创新，单单架构简单也是有技术优势的，可以作为参考 实现更底层的 c语言实现。
3. verysimple_c 写好后，就可以尝试将 naiveproxy 嵌入 verysimple_c 了
4. 完善 tls lazy encrypt技术
5. 链接池技术，可以重用与服务端的连接 来发起新请求
6. 握手延迟窗口技术，可用于分流一部分流量使用mux发送，达到精准降低延迟的目的；然后零星的链接依然使用单独信道。


verysimple 继承 v2simple的一个优点，就是服务端的配置也可以用url做到。谁规定url只能用于分享客户端配置了？一条url肯定比json更容易配置，不容易出错。

不过，显然url无法配置大量复杂的内容，而且有些玩家也喜欢一份配置可以搞定多种内核，所以未来 verysimple 会推出兼容 v2ray的json配置 的模块。**只是兼容配置格式，不是兼容所有协议！目前只有vless v0 协议是兼容的，可以直接使用现有其它客户端**

其它开发计划请参考
https://github.com/hahahrfool/v2ray_simple/discussions/3

## 生成自签名证书

注意运行第二行命令时会要求你输入一些信息。确保至少有一行不是空白即可，比如打个1
```sh
openssl ecparam -genkey -name prime256v1 -out cert.key
openssl req -new -x509 -days 7305 -key cert.key -out cert.pem
```

我给出的命令会生成ecc证书，这个证书速度更快, 有利于网速加速（加速tls握手）。

不要在实际场合使用我提供的证书！自己生成！而且最好是用 自己真实拥有的域名，使用acme.sh等脚本申请免费证书，特别是建站等情况。

使用自签名证书是会被中间人攻击的，再次特地提醒。如果被中间人攻击，就能直接获取你的uuid，然后你的服务器 攻击者就也能用了。

仅有ip是不够的，要结合域名。本项目提供的自签名证书仅供快速测试使用，切勿用于实际场合。
## 测速

测试环境：ubuntu虚拟机, 使用开源测试工具
https://github.com/librespeed/speedtest-go

编译后运行，会监听8989。注意要先按speedtest-go的要求，把web/asset文件夹 和一个toml配置文件 放到 可执行文件的文件夹中,我们直接在项目文件夹里编译的，所以直接移动到项目文件夹根部即可

然后内网搭建nginx 前置，加自签名证书，配置添加反代：
`proxy_pass http://127.0.0.1:8989;`
然后 speedtest-go 后置。

然后verysimple本地同时开启 客户端和 服务端，然后浏览器 firefox配置 使用 socks5代理，连到我们的verysimple客户端

注意访问测速网页时要访问https的，否则测的 splice的速度实际上还是普通的tls速度，并没有真正splice。

访问 htts://自己ip/example-singleServer-full.html
注意这个自己ip不能为 127.0.0.1，因为本地回环是永远不过代理的，要配置成自己的局域网ip。

### 结果

左侧下载，右侧上传，单位Mbps。我的虚拟机性能太差，所以就算内网连接速度也很低。

不过这样正好可以测出不同代理协议之间的差距。

```
//直连
156，221
163，189
165，226
162，200


//verysimple, vless v0 + tls
145，219
152，189
140，222
149，203

//verysimple, vless v0 + tls + tls lazy encrypt (splice):

161，191，
176，177
178，258
159，157
```

详细测速还可以参考另外两个文件，speed_macos.md 和 speed_ubuntu.md。

总之目前可以看到，在有splice的情况下（即linux中），verysimple是绝对的王者。虽然有时还不够稳定，但是我会进一步优化这个问题的。

## 交流

https://t.me/shadowrocket_unofficial


# 免责声明与鸣谢

## 免责

MIT协议！我不负任何责任。本项目只是个代理项目，适合内网测试使用，以及适合阅读代码了解原理。

你如果用于任何其它目的，我不会帮助你。

我只会帮助研究理论的朋友。而且我不帮你你也没话说，MIT协议。

同时，我们对于v2ray/xray等项目也是没有任何责任的。
