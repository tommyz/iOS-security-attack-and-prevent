## iOS安全攻与防

* 本地数据攻与防* https* Uiwebview* 第三方sdk与xcode* 反编译与代码混淆* 越狱与反调试* 扫描工具fortify* 常见接口漏洞分析### 本地数据攻与防

#### APP文件下的本地存储Documents、Library/Caches、Tmp* Documents: 保存应⽤运行时生成的需要持久化的数据,iTunes同步设备时会备份该目录。
* tmp: 保存应⽤运行时所需的临时数据,使⽤完毕后再将相应的文件从该目录删除。应用没有运行时,系统也可能会清除该目录下的文件。iTunes同步设备时不会备份该目录。
* Library/Caches: 保存应用运行时⽣成的需要持久化的数据,iTunes同步设备时不会备份该目录。一般存储体积大、不需要备份的非重要数据，比如网络数据缓存存储到Caches下。


如下图所示在越狱设备中，完全可以利用iTools等其他工具将这些数据导出。![图片1](/Users/shenweishun264/Documents/iOS安全攻与防/图片 1.png)#### 防止数据泄露建议

1.对于主动存储在app内的重要的、有价值的、涉及隐私的信息需要加密处理，增加攻击者破解难度。2.对于一些非主动的存储行为如网络缓存，涉及重要信息，做到用完即删。#### 数据库（sqlite）的安全问题

APP一般运用数据库存储一些数据量比较大，逻辑比较复杂的数据，方便我们使用和增加使用效率。数据库，一般存储在Documents路径下的特殊文件，格式一般为.db或者.sqlite，导出后使用特殊工具即可查看如DB Browser for sqlite 和 SQLiteStudio。**防止攻击者读取数据库的建议:**

**数据加密：** 使用如AES256加密算法对数据进行安全加密后再存入数据库中。          优点：使用简单，无需第三方库支持。          缺点：每次存储都有加密解密过程，增加APP资源消耗。**整库加密：** 可使用第三方的SQLite扩展库，对数据库进行整体的加密。如：SQLCipher，[git地址](https://github.com/sqlcipher/sqlcipher.git)                           优点：对数据库整体操作，减少资源消耗。          缺点：需要使用第三库。                        
在创建数据库时，添加如下代码即可：

![图片2](/Users/shenweishun264/Documents/iOS安全攻与防/图片 2.png)

这时再打开数据库会弹出输入密码窗口，只有输入密码方可打开：

![图片3](/Users/shenweishun264/Documents/iOS安全攻与防/图片 3.png)

**sqlcipher使用demo请参考:**[数据库sqlite3](https://github.com/tianjifou/CoreSQLite3)
#### KeyChain数据的读取

Keychain是一个拥有有限访问权限的SQLite数据库（AES256加密），可以为多种应用程序或网络服务存储少量的敏感数据（如用户名、密码、加密密钥等）。如保存身份和密码，以提供透明的认证，使得不必每次都提示用户登录。在iPhone上，Keychain所存储的数据在     /private/var/Keychains/keychain-2.db SQLite数据库中。如下图：

![图片4](/Users/shenweishun264/Documents/iOS安全攻与防/图片 4.png)

当我们打开这个数据库，会发现如下图中四个表：genp、inet、cert、keys![图片5](/Users/shenweishun264/Documents/iOS安全攻与防/图片 5.png)
                    
分别对应下图中的前四列，下图表示iOS系统的keychain 存储类型

![图片6](/Users/shenweishun264/Documents/iOS安全攻与防/图片 6.png)            数据库内数据，大多数是加密的，Keychain的数据库内容使用了设备唯一的硬件密钥进行加密，该硬件密钥无法从设备上导出。因此，存储在Keychain中的数据只能在该台设备上读取，而无法复制到另一台设备上解密后读取。![图片7](/Users/shenweishun264/Documents/iOS安全攻与防/图片 7.png)

一旦攻击者能够物理接触到没有设置密码的iOS设备时，他就可以通过越狱该设备，运行如keychain_dumper这样的工具，读取到设备所有的Keychain条目，获取里面存储的明文信息。具体方法是通过ssl让mac连接iPhone，使用keychain_dumper，导出Keychain。具体步骤如下：1. 将手机越狱，通过Cydia（越狱手机都有，相当于App Store）安装OpenSSH。
2. 在mac终端输入：  ssh root@（手机IP）  然后会提示输入密码，默认为alpine
3. 使keychain数据库权限可读：    cd /private/var/Keychains/   chmod +r keychain-2.db
   4. 下载工具Keychain-Dumper [git地址](https://github.com/ptoomey3/Keychain-Dumper)
5. 将下载的keychain_dumper可执行文件移到iPhone的/bin目录下
 * Ctrl+D，退出当前ssh连接 * 输入命令：scp /Users/ice/Downloads/Keychain-Dumper-master/keychain_dumper root@（手机ip）:/bin/keychain_dumper6. 添加执行权限:  chmod +x /bin/keychain_dumper7. 解密keychain：/bin/keychain_dumper最后输出如下图所示的手机应用内所有使用keychain的情况，下图展示的是一个用户存储的WiFi账号和密码。![图片8](/Users/shenweishun264/Documents/iOS安全攻与防/图片 8.png)

综上可知，无论以何种方式在手机内存储数据，攻击者总是有办法获取到，只是不同的方式，攻击者获取的难度不一样，从这个角度来说keychain还是比较安全的存储方式，但是还是需要加密隐私信息。这也说明了我们对缓存数据加密的必要性。#### 剪切板的缓存

我们在使用剪切板进行操作时，iOS系统会缓存所操作的数据，如果数据涉及隐私，将会带来安全问题。如下图，iOS的三种剪切板和功能，其中系统级别的剪切板优先级最高。![图片9](/Users/shenweishun264/Documents/iOS安全攻与防/图片 9.png)

注意：在获取剪切板内容时，总是先取系统的剪切板内容，若没有值才会取应用间的剪切板内容。所以在使用完剪切板时，我们在程序退出时，清空剪切板内容。![图片10](/Users/shenweishun264/Documents/iOS安全攻与防/图片 10.png)#### 系统键盘缓存

我们打字在使用系统键盘打字时时，会出现提示，键盘一般会缓存这些提示字符串，缓存在手机的“/private/var/mobile/Library/Keyboard/dynamic-text.dat”文件中，攻击者可以提取出来，出现频率最高的往往是账户名和密码。如下图所示：![图片11](/Users/shenweishun264/Documents/iOS安全攻与防/图片 11.png)**系统键盘不会缓存条件：**

1. 禁用了自动纠正功能的文本框会阻止输入内容被缓存。   设置UITextView/UITextField 属性autocorrectionType 为. no类型 2. 非常短的输入，如只有1或者2个字母组成的单词不会被缓存。3. 输入框被标记为密码输入类型，也不会被缓存。   设置UITextView/UITextField 属性isSecureTextEntry为true4. iOS 5之后，输入只包括数字的字符串不会被缓存。    设置UITextView/UITextField属性 keyboardType为数字键盘。5. 应用内自定义键盘，银行类APP大多使用自定义键盘。


**第三方键盘权限问题：**

第三方键盘可能会收集自己的信息以及上传自己的信息，因此在苹果设置里禁止这种行为至关重要，再有就是，因为第三方键盘不可控因素很多，可能缓存数据逻辑与系统键盘不一样，防止攻击者攻击本地缓存数据，因此建议尽量少用第三方键盘。如下图，苹果给的建议。另外，我们使用的第三方键盘，会上传你的键入数据到服务器，这样我们的隐私完全暴露给第三方，使用时因注意不要启用不完全访问。![图片12](/Users/shenweishun264/Documents/iOS安全攻与防/图片 12.png)#### 应用挂起时的截屏行为

当应用被挂起时，如切到后台或者接电话等，系统会自动在当前应用的页面截屏并存储到手机内，如果当前页面设计敏感信息时，被攻击会造成泄密。如下图生成的两张图片路径：![图片13](/Users/shenweishun264/Documents/iOS安全攻与防/图片 13.png)

**防止手机截屏图片泄密方案：**

一般我们在应用被挂起时，在当前页面添加一层高斯模糊，在应用重新进入前台时，删除模糊效果，代码如下，截屏图片效果如下图：

![图片14](/Users/shenweishun264/Documents/iOS安全攻与防/图片 14.png)**代码如下：**

![图片15](/Users/shenweishun264/Documents/iOS安全攻与防/图片 15.png)

![图片16](/Users/shenweishun264/Documents/iOS安全攻与防/图片 16.png)

### HTTPS问题HTTPS协议其实是有两部分组成：HTTP + SSL / TLS，也就是在HTTP上又加了一层处理加密信息的模块。服务端和客户端的信息传输都会通过TLS进行加密，所以传输的数据都是加密后的数据，具体流程如下图所示：![图片17](/Users/shenweishun264/Documents/iOS安全攻与防/图片 17.png)1. 客户端将自己支持的一套加密规则发送给服务端。 2. 服务端从中选出一组加密算法与HASH算法，并将自己的身份信息以证书的形式发回给客户端。证书里面包含了服务端地址，加密公钥，以及证书的颁发机构等信息。 3. 客户端获得服务端证书之后客户端要做以下工作： 
  a) 验证证书的合法性（颁发证书的机构是否合法，证书中包含的服务端地址是否与正在访问的地址一致等）。
  b) 如果证书受信任，或者是用户接受了不受信的证书，客户端会生成一串随机数的密码，并用证书中提供的公钥加密。 
    c) 使用约定好的HASH算法计算握手消息，并使用生成的随机数对消息进行加密，最后将之前生成的所有信息发送给服务端。 4. 服务端接收客户端发来的数据之后要做以下的操作： 
  a) 使用自己的私钥将信息解密取出密码，使用密码解密客户端发来的握手消息，并验证HASH是否与客户端发来的一致。 
  b) 使用密码加密一段握手消息，发送给客户端。 5. 客户端解密并计算握手消息的HASH，如果与服务端发来的HASH一致，此时握手过程结束，之后所有的通信数据将由之前客户端生成的随机密码并利用对称加密算法进行加密。**HTTPS存在的漏洞：**

  一、SSL证书伪造
  
  首先通过ARP欺骗、DNS劫持甚至网关劫持等等，将客户端的访问重定向到攻击者的机器，让客户端机器与攻击者机器建立HTTPS连接（使用伪造证书），而攻击者机器再跟服务端连接。这样用户在客户端看到的是相同域名的网站，但客户端会提示证书不可信，用户只要做了证书验证就能避免被劫持的。![图片18](/Users/shenweishun264/Documents/iOS安全攻与防/图片 18.png)

 二、SSL剥离攻击
 
 该攻击方式主要是利用用户并不会每次都直接请求https://xxx.xxx.com来访问网站，或者有些网站并非全网HTTPS，而是只在需要进行敏感数据传输时才使用HTTPS的漏洞。中间人攻击者在劫持了客户端与服务端的HTTP会话后，将HTTP页面里面所有的https://超链接都换成http://，用户在点击相应的链接时，是使用HTTP协议来进行访问；这样，就算服务器对相应的URL只支持HTTPS链接，但中间人一样可以和服务建立HTTPS连接之后，将数据使用HTTP协议转发给客户端，实现会话劫持。![图片19](/Users/shenweishun264/Documents/iOS安全攻与防/图片 19.png)
三、 Charles等第三方抓包工具

抓包工具可以通过截获真实客户端的HTTPS请求，伪装客户端向真实服务端发送HTTPS请求和接受真实服务器响应，用自己的证书伪装服务端向真实客户端发送数据内容，我们将私有CA签发的数字证书安装到手机中并且作为受信任证书保存。下图所示就是Charles的可信任证书。想了解Charles如何抓包的请自行Google，网上有很多文章，在此不过多介绍。

![图片20](/Users/shenweishun264/Documents/iOS安全攻与防/图片 20.png)

**防范https漏洞：**

我们使用AFNetworking网络框架来配置证书验证，而我们需要配置这个框架下的类AFSecurityPolicy相关属性即可。在此我们使用AFSSLPinningModeCertificate对证书进行完整校验。

![图片21](/Users/shenweishun264/Documents/iOS安全攻与防/图片 21.png)
AFSSLPinningModeNone: 代表客户端无条件地信任服务器端返回的证书。
如果选择此种证书类型，由于客户端没有做任何的证书校验，所以此时随意一张证书都可以进行中间人攻击，可以使用burp里的这个模块进行中间人攻击。

![图片23](/Users/shenweishun264/Documents/iOS安全攻与防/图片 23.png)
AFSSLPinningModePublicKey: 代表客户端会将服务器端返回的证书与本地保存的证书中，PublicKey的部分进行校验；如果正确，才继续进行。

如果选择此种类型证书的话，客户端只是做了部分校验，例如在证书校验过程中只做了证书域名是否匹配的校验，可以使用burp的如下模块生成任意域名的伪造证书进行中间人攻击。![图片24](/Users/shenweishun264/Documents/iOS安全攻与防/图片 24.png)

如果客户端对证书链做了校验，那么攻击难度就会上升一个层次，此时需要人为的信任伪造的证书或者安装伪造的CA公钥证书从而间接信任伪造的证书，可以使用burp的如下模块进行中间人攻击。![图片25](/Users/shenweishun264/Documents/iOS安全攻与防/图片 25.png)
AFSSLPinningModeCertificate: 代表客户端会将服务器端返回的证书和本地保存的证书中的所有内容，包括PublicKey和证书部分，全部进行校验；如果正确，才继续进行。这种方式可以避免中间人攻击。一般做法是将服务器生成的证书打包，导入客户端内，使用包的二进制数据，进行全部校验，但是，打包进app会有被攻击者反编译出的风险，一般做法是将二进制数据转换成base64的字符串，存入代码内。

注意：服务器给的证书一般是.crt格式的，我们需要转化成.cer格式，终端执行命令：openssl x509 -in 证书name.crt -out 证书name.cer -outform der。![图片26](/Users/shenweishun264/Documents/iOS安全攻与防/图片 26.png)
![图片22](/Users/shenweishun264/Documents/iOS安全攻与防/图片 22.png)


因此配置 allowInvalidCertificates = false, validatesDomainName = truepinnedCertificates 属性是证书的二进制数据，对于证书来说，一般是.cer格式的文件，拖入项目中，然后读取二进制数据对其赋值，但是，项目中的文件，攻击者很容易通过解包取出，所以我们一般，在代码中存放证书的二进制流数据

### UIWebView按漏洞**一、 UIWebView XSS漏洞**（1） 未知的来源的进入了web页面，如对用户输入框输入的数据没有进行过滤，恶意数据通过用户组件，URL scheme handlers或者notifications传递给了UIWebView组件，并且UIWebView组件没有验证该数据的合法性。（2 ） UIWebView通过loadHTMLString:baseURL：这个方法从本地html读取代码，然后加载，而这个HTML代码并没有被信任的未知来源，这个恶意的本地HTML可以读取所有的保存在该应用里的cookie，绕过同源策略做一些非法的操作，攻击者将用户的像cookie，session等敏感数据传递给攻击者，或者重定向用户页面到攻击者控制的服务器上，或者诱导用户输入一些敏感数据，然后传递到攻击者的服务器上，实现钓鱼攻击。
**方案：**（1）对输入框内的数据进行关键词过滤（如“script”字符串），避免输入恶意脚本语言。（2）在加载网页的内容之前，会用dataWithContentsOfURL方法将加载的网页的内容提取出来放入NSData中，在使用loadHTMLString之前可以对NSData的内容过滤，编码等等，上述代码中是将script字符串都替换掉，保证了UIWebView不去加载脚本，从而也避免了加载恶意脚本的可能，如下代码：![图片27](/Users/shenweishun264/Documents/iOS安全攻与防/图片 27.png)

 或者可以通过这些截断的URL来做一些白名单、黑名单策略，如禁用file域，不让它加载本地文件![图片28](/Users/shenweishun264/Documents/iOS安全攻与防/图片 28.png)

**二、UIWebView https中间人攻击漏洞**    UIWebView中间人漏洞有如下两种情况，第一种情况是直接没有没有校验证书合法性，如校验根证书是否为自签名，是否可信，证书是否过期。第二种情况是校验了证书的合法性但没有校验访问的主机域名与证书上的域名是否匹配。通过中间人攻击可以查看客户端与服务器端传输的数据，有可能导致用户隐私数据的泄露，通过中间人可以修改客户端与服务器之间传递的数据，有可能导致用户财产的损失。**解决方案：**在UIWebViewDelegate的代理方法中拦截请求，并用AFNetworing框架对该请求进行https验证，验证通过再加载网页，代码如下：
 
![图片29](/Users/shenweishun264/Documents/iOS安全攻与防/图片 29.png)
但是，我们的app很难做到全是https，域名验证就无法通过，UIWebView的https证书验证添加还有待讨论。### Xcode与第三方库来源

如下图所示,2015年的xcode后门事件，原因就是从非官方渠道下载的xcode，被导入恶意代码会上传产品自身的部分基本信息，包括安装时间、应用id，应用名称、系统版本以及国家和语言等。

![图片30](/Users/shenweishun264/Documents/iOS安全攻与防/图片 30.png)
**验证Xcode来源：****终端内输入：**spctl --assess --verbose/Applications/Xcode.app 
如果输出是/Applications/Xcode.app: accepted source=Apple 或者

/Applications/Xcode.app: accepted  source=Apple System 或者

/Applications/Xcode.app: accepted source=Mac App Store 中的一个，表明来源可靠。

因此，我们为防止工具和第三方库被污染，使用时一定要注意来源，对于SDK来说，选用之前要做到观看代码逻辑，避免恶意代码，对于含有动态库或者静态库的SDK，尽量避免使用，或者运用反编译手段，逆向SDK，这点难度较大。### 逆向工程反编译APP市场上有很多反编译工具如class-dump-z、IDA、Hopper Disassembler，他们通常是和Clutch一起使用。因为APP Store上的APP都是加密过的，需要先解密，解密一般可以得到资源文件、二进制文件， Clutch 就是做这个工作的软件。class-dump-z ，一般用来反编译oc工程，不支持swift，IDA收费，而且太贵，我们主要使用Hopper来反编译项目。只要将项目的二进制文件拖入hopper内即可反编译。下图就是反编译，我们的项目中的配置的key值。![图片31](/Users/shenweishun264/Documents/iOS安全攻与防/图片 31.png)
下图是反编译出的项目中的所有方法：![图片32](/Users/shenweishun264/Documents/iOS安全攻与防/图片 32.png)
下图是随机抽取到的PACreateFollowUpViewController的ViewDidLoad方法的汇编实现形式，大致逻辑可见。![图片33](/Users/shenweishun264/Documents/iOS安全攻与防/图片 33.png)

下图是反编译出的项目中使用的字符串内容：![图片34](/Users/shenweishun264/Documents/iOS安全攻与防/图片 34.png)
**综上所述，逆向工程师可以通过Hopper软件破解APP，观看里面的算法逻辑，或者利用工具向包内添加代码更改原来的逻辑。解决方案：****字符串加密，类名方法名、代码块混淆**
**一、字符串加密：**

 **现状：** 对于字符串来说，程序里面的明文字符串给静态分析提供了极大的帮助，比如说根据界面特殊字符串提示信息，从而定义到程序代码块，或者获取程序使用的一些网络接口等等。
 
 **加固：**对程序中使用到字符串的地方，首先获取到使用到的字符串，当然要注意哪些是能加密，哪些不能加密的，然后对字符串进行加密，并保存加密后的数据，再在使用字符串的地方插入解密算法，这样就很好的保护了明文字符串。
 
 **二、类名方法名混淆：**
 **现状：**目前市面上的IOS应用基本上是没有使用类名方法名混淆的，所以只要我们使用class-dump把应用的类和方法定义dump下来，然后根据方法名就能够判断很多程序的处理函数是在哪。从而进行hook等操作。
  

 **加固：**对于程序中的类名方法名，自己产生一个随机的字符串来替换这些定义的类名和方法名，但是不是所有类名，方法名都能替换的，要过滤到系统有关的函数以及类。**三、程序代码混淆：**
  **现状：**目前的IOS应用找到可执行文件然后拖到Hopper Disassembler或者IDA里面程序的逻辑基本一目了然。   **加固：**可以基于Xcode使用的编译器clang，然后在中间层也就是IR实现自己的一些混淆处理，比如加入一些无用的逻辑块啊，代码块啊，以及加入各种跳转但是又不影响程序原有的逻辑。可以参考下开源项目， 当然开源项目中也是存在一些问题的，还需自己再去做一些优化工作。   **四、加入安全SDK：**
      **现状：**目前大多数IOS应用对于简单的反调试功能都没有，更别说注入检测，以及其它的一些检测了。   **加固：**加入SDK，包括多处调试检测，注入检测，越狱检测，关键代码加密，防篡改等等功能。并提供接口给开发者处理检测结果。
   
   
### Cycript脚本语言显示和修改APP的UI布局与代码逻辑Cycript是由Cydia创始人Saurik推出的一款脚本语言,支持oc和js语言的编辑器。通过Cycript下载安装可以直接使用。Cycript [语法使用文档](http://iphonedevwiki.net/index.php/Cycript_Tricks)

**Cycript 作用：**

1. 创建对象
2. 输出当前APP页面布局（UIApp.keyWindow.recursiveDescription().toString()）
3. 添加方法
4. 更改对象属性
5. Load frameworks
...   **Cycript 使用示例：**

1. 通过Cydia下载安装Cycript
2. 通过ssh连接手机和电脑
   在mac终端输入：  ssh root@（手机IP）  然后会提示输入密码，默认为alpine
3. 输出所要更改应用的当前进程地址和文件地址：    ps aux | grep demo名（不一定是APP名）    输出如下结果表示成功了![图片35](/Users/shenweishun264/Documents/iOS安全攻与防/图片 35.png)

4.  使用cycript连接这个app   cycript -p 6542   输出 cy# 表示成功 即可输出相关指令更改APP了
   
**例如打印当前页面的布局信息：**

![图片36](/Users/shenweishun264/Documents/iOS安全攻与防/图片 36.png)


详细使用方式，请参考[cycript语法文档](http://iphonedevwiki.net/index.php/Cycript_Tricks)在此不过多介绍


**Cycript危害：**

使用cycript和反编译工具是逆向APP工程的基本工具，cycript是解析APP布局和修改代码逻辑的利器，为防止cycript修改代码逻辑，从而逃过我们自己代码的逻辑，我们在编写重要流程时，需要提醒接口也要鉴权判断，如支付订单时，在钱不够的情况下，我们不让button可以点击，但是通过cycript，我们可以修改代码button的属性enable = YES，这样如果后台不做判断导致支付成功，会引发安全问题。### 检测越狱与反调试

很多APP漏洞问题都是基于设备越狱的条件，因此在完成某些行为时，检测APP是否越狱，可以有效的防止攻击者的破解行为。**1.常见越狱存在的文件**      /Applications/Cydia.app      /Library/MobileSubstrate/MobileSubstrate.dylib      /bin/bash      /usr/sbin/sshd      /etc/apt   在下面代码中输入上述路径，判断是否存在，存在表示该设备已被越狱![图片37](/Users/shenweishun264/Documents/iOS安全攻与防/图片 37.png)

**2.判断cydia的URL scheme**  一般越狱的设备中都会安装cydia这个应用，我们可以通过是否可以打开这个应用检测设备是否越狱。 ![图片38](/Users/shenweishun264/Documents/iOS安全攻与防/图片 38.png)
  
#### 反调试我们可以使用<sys/types.h>和<dlfcn.h>库来阻止GDB依附，从而达到阻止攻击者破解代码后，对我们的程序进行调试。具体代码实现如下图： ![图片39](/Users/shenweishun264/Documents/iOS安全攻与防/图片 39.png)

### 扫描工具对代码的扫描市面上有很多扫描工具，一般比较流行的有fortify和coverity，其中coverity对代码的检查侧重于代码质量，fortify侧重于代码的安全漏洞。我们这里介绍的是使用fortify对代码的漏洞分析。如何使用fortify,SVN已有，扫描结果如下： ![图片40](/Users/shenweishun264/Documents/iOS安全攻与防/图片 40.png)

**扫描结果分析：**

一、发现一个关于 ATS（App Transport Security）问题，iOS9中新增App Transport Security（简称ATS）特性, 主要使到原来请求的时候用到的HTTP，都转向TLS1.2协议进行传输，也就是https，通过在 Info.plist 中添加 NSAppTransportSecurity 字典并且将 NSAllowsArbitraryLoads 设置为 YES 来禁用 ATS，但是这种行为是不安全的，也不是苹果推荐的，结合我们使用需求，很多第三方库和网页还是使用http，所以就没处理。二、在info.plist内配置了一个APIkey值，info.plist里的内容是完全可以暴露出来的，在里面配置key值，也是一种不安全的行为。这个key是项目中bug统计Fabric脚本自动添加的key，目前已修正。### 项目中的接口常用漏洞：一、医生权限登录，意见反馈／资质审核等处，添加照片未对上传文件后缀名作限制，可实现任意文件上传。 ![图片41](/Users/shenweishun264/Documents/iOS安全攻与防/图片 41.png)
这个是上传图片接口，上传时并没有对上传的图片二进制流数据进行格式判断，攻击者可以通过抓包更改数据流，例如，将图片二进制流替换成含有病毒的HTML的文件数据流，我们访问时，会受影响。

二、下单时修改陪诊服务费参数，将费用由99.0篡改为0.1，提交后成功下单 ![图片42](/Users/shenweishun264/Documents/iOS安全攻与防/图片 42.png)
针对支付漏洞，后端应严格校验订单对应的金额，防止攻击者进行篡改价格提交接口，引发重大漏洞。三、万家医疗app采用了https,但对证书校验不当，可通过信任bupsuite等代理工具拦截到https通信数据 ![图片43](/Users/shenweishun264/Documents/iOS安全攻与防/图片 43.png)

我们使用AFNetworking网络框架请求接口，然而并未对证书进行比对校验，使得只要是ca颁发的的可信任证书，都可以验证通过，从而导致中间人攻击或者被抓包。四、账号1在查询自己订单时将订单参数wanjiaOrderId替换为账号2的值即可查询账号2的订单

 ![图片44](/Users/shenweishun264/Documents/iOS安全攻与防/图片 44.png)

在查询账号1随访记录时通过拦截数据包修改clinicId参数为账号2的对应的值即可实现越权查询 ![图片45](/Users/shenweishun264/Documents/iOS安全攻与防/图片 45.png)
越权漏洞的成因主要是因为开发人员在对数据进行增、删、改、查询时对客户端请求的数据过分相信而遗漏了权限的判定。大多数应用程序在功能在UI中可见以前，验证功能级别的访问权限。但是，应用程序需要在每个功能被访问时在服务器端执行相同的访问控制检查。如果请求没有被验证，攻击者能够伪造请求以在未经适当授权时访问功能。因此应该严格后端严格校验参数与用户的所属关系，防止越权。五、账号退出登录后相关接口依然可以操作 ![图片46](/Users/shenweishun264/Documents/iOS安全攻与防/图片 46.png)
添加退出登录接口，退出登录时，立即注销session。防止退出登录后，依然可以通过接口请求数据。六、 APP登录处，选择动态密码登录，登录成功后直接返回了用户账号密码对应的md5值

 ![图片47](/Users/shenweishun264/Documents/iOS安全攻与防/图片 47.png)

所有接口不应该返回用户账号密码字段，应该屏蔽掉，即使返回的是MD5加密过的，也有可能被破解。七、相关订单查询接口，请务必统一整改，涉及敏感信息：身份证号,手机号等未做脱敏处理 
 ![图片48](/Users/shenweishun264/Documents/iOS安全攻与防/图片 48.png)

在接口中必须返回一些敏感信息时，应该使用对称加密方式加密处理。防止攻击者盗用用户隐私信息。八、登录会话一天未操作后进入app会话依然有效，为保证用户账号安全，应建立会话失效机制

 ![图片49](/Users/shenweishun264/Documents/iOS安全攻与防/图片 49.png)
每次请求接口，应该校验token值是否在规定时间内有效，无效则强制退出,在一定程度上防止攻击者使用抓到的包或者接口一直恶意请求服务器。