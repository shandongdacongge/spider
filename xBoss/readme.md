> 近日出于学习的目的对某Boss网站的反爬机制进行了分析和逆向，终于完全搞定了，记录总结下方便日后学习！
本代码请仅用于 纯技术研究的 用途，请勿用于非法用途，如果因使用者非法使用造成的法律问题与本作者无关！

# 访问流程
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

> 对于2或3 拿到seed、name、ts后的流程基本一样，都是调用new ABC().prototype.z()函数，生成 __zp_stoken__ 并做一次encodeURIComponent后存放到cookie中。

# 参数说明
- seed 是服务端发送的一个种子，由服务端控制生成
- name 是对应加密js的名字，这个name在每次访问时会动态变化，也及时获取的加密js不固定，由服务端控制生成，8位数字或字母的组合
- ts 是timestamp时间戳，由服务端控制生成
- __zp_stoken__ 通过name指定的加密js 计算出来 cookie值，由客户端基于服务器提供seed和ts来生成，每次发送请求需携带有效的 __zp_stoken__

第一次访问流程
第一次访问不携带token，返回一个302跳转到，注意这个302地址中的三个参数，seed ts name非常重要，其中name就是此次动态js的名字，302跳转后的页面会按这个参数加载核心js文件
302页面分析
生成token后再次访问真正页面
后续访问流程
每次正常访问后页面响应会设置三个cookie值，seed ts name 
