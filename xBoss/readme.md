> 近日出于学习的目的对某Boss网站的反爬机制进行了分析和逆向，终于完全搞定了，记录总结下方便日后学习！
本代码请仅用于 纯技术研究的 用途，请勿用于非法用途，如果因使用者非法使用造成的法律问题与本作者无关！

> 作者已经把该网站的核心js加密文件进行了全逆向，爬取方案不需要浏览器参与，纯python+js即可！

# 涉及技术
本爬取方案最终通过Python实现了完全自动化爬取职位信息列表页，并存储到一个txt文件中(需要的可自行扩展数据库实现)，为了简单起见未使用多线程，使用的技术有：
- python
- requests 
- execjs 执行js脚本
- re 正则表达式
- BeautifulSoup 解析列表页面提取职位信息
- jsbeautify 为增加可读性，对加密js进行格式化处理

# 爬取方案流程
爬取方案的思路为
1. 访问页面获取302跳转的Location，提取seed name ts核心参数
2. 根据name请求对应加密js文件
3. 使用jsbeautify对加密js文件进行格式化
4. 对加密js进行运算法替换以及解毒处理
5. 调用js生成 __zp_stoken__
6. 通过 __zp_stoken__ 爬取指定列表页数据
7. 解析页数，提取职位信息追加到result.txt中
8. 解析页面响应的头部信息，获取seed name ts
9. 根据seed name ts 再次 从第二步开始重复 

# 网站正常访问流程
1. 某Boss网站在访问页面时都需要携带一个 __zp_stoken__=xxxxxx 这样的cookie
2. 每次访问页面都会更新这个__zp_stoken__，（另外，每次__zp_stoken__其实可以使用数次（貌似<5次），具体次数没有试验）
    - 返回的页面会携带 __zp_sseed__、 __zp_sname__、__zp_snts__ 三个cookie （通过查看response 的 set-cookie部分）
    - 基于这三个cookie值计算新的__zp_stoken__存入cookie中
    - 这个是三个cookie值消费完后会即刻删除，所以在浏览器中一般看不到这三个cookie值
3. 如果不携带cookie __zp_stoken__或者__zp_stoken__无效，就会返回一个302跳转
    - 302跳转Location 为  https://www.zhipin.com/web/common/security-check.html?seed=r9L%2BjsS5vloxpthkin%2F2Yskecz1G198Nk%2B4RCkA2YiA%3D&name=d05572cd&ts=1587092342543&callbackUrl=%2Fc101280600%2F%3Fquery%3Dpython%26page%3D2%26ka%3Dpage-2&srcReferer=
    - Location 中seed、name、ts为关键参数，用于计算 __zp_stoken__
    - 这个页面中的js所做的工作就是为
        - 按照name加载核心加密js文件
        - 设置cookie中的 __zp_stoken__
        - 精简后的核心js为
        ```javascript
                    var getQueryString = function(name) {
                        var reg = new RegExp("(^|&)" + name + "=([^&]*)(&|$)");
                        var r = window.location.search.substr(1).match(reg);
                        if (r != null) return unescape(r[2]);
                        return null;
                    };
					
					var seed = decodeURIComponent(getQueryString("seed")) || "";
                    var ts = getQueryString("ts");
                    var fileName = getQueryString("name");
					
					var expiredate = new Date().getTime() + 32 * 60 * 60 * 1000 * 2;
					var code = "";
					var ABC = window.ABC || frame.contentWindow.ABC;
					try {
						//code = new ABC().z(seed, parseInt(ts)+(480+new Date().getTimezoneOffset())*60*1000);
						code = new ABC().z(seed, parseInt(ts));//+(480+new Date().getTimezoneOffset())*60*1000 时区相对于北京的偏移
					} catch (e) {}
					if (code ) Cookie.set("__zp_stoken__", code, expiredate, COOKIE_DOMAIN, "/"); //该方法会做一次 encodeURIComponent
        ```

> 对于2或3 拿到seed、name、ts后的流程基本一样，都是调用new ABC().prototype.z(seed,ts)函数，生成 __zp_stoken__ 并做一次 encodeURIComponent 后存放到cookie中。

# 术语说明
- seed 是服务端发送的一个种子，由服务端控制生成
- name 是对应加密js的名字，这个name在每次访问时会动态变化，也及时获取的加密js不固定，由服务端控制生成，8位数字或字母的组合
- ts 是timestamp时间戳，由服务端控制生成
- __zp_stoken__ 通过name指定的加密js 计算出来 cookie值，由客户端基于服务器提供seed和ts来生成，每次发送请求需携带有效的 __zp_stoken__
- 投毒 js代码中会对环境进行很多检测，如果识别为非正常浏览器，就会对某些参数进行修改，造成最终结果错误，成为投毒
- 解毒 对代码中的投毒代码进行处理
- zz 加密js中的一个变量，对应最终生成的 __zp_stoken__ ，是难得一见的具备可读性的变量
- ABC 加密js中的一个对象，也是生成最终 __zp_stoken__ 的入口
- 常量池 加密js中的第一行，就是定义了一个常量字符串数组（base64加密），其中存放的都是代码中用到字符串
- 常量池偏移 常量池在进行字符串替换时，并不是直接按下标替换的，而是做了一个偏移（不同的js不一样）
- 运算符函数 就是将一些简单的运算符转成了函数 如+-*/等等，原始js中存在大量的运算符函数
- 运算符函数替换 对js中的运算符函数 替代成原始 操作符，并干掉运算符函数的过程就是运算符函数替换
- switch扁平化 原始js中很多switch case语句用来控制函数的执行流程，而判断的内容来源于某个数组，这个数组又是根据常量池中的某个常量按照|进行分割形成的，其实switch最终执行流程是固定，而按照这个数据对switch case进行函数顺序编排及替换就成为 switch扁平化

# 相关代码文件说明
