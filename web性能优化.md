# 1.资源合并与压缩

**方法:**
1. 合并请求减少http数
2. 减小请求文件大小

**工具**

fis3、webpack、gulp等自动化构建工具

# 2.图片资源优化

**概念**
1. png8/png24/png32之间的区别
	- png8:256色+支持透明
	- png24:2的24次方色+不支持透明
	- png32:2的24次方色+支持透明
	- 每种图片格式都有自己的特点,针对不同的业务场景选择不同的图片格式
2. 不同格式图片常用的业务场景
	- jpg有损压缩,压缩率高,不支持透明:(大部分不需要透明图片的业务场景)
	- png支持透明,浏览器兼容好:(大部分需要透明图片的业务场景)
	- webp压缩程度更好,在ios webview有兼容问题:(安卓全部)
	- svg矢量图,代码内嵌,相对较小,图片样式相对简单的场景:(图片相对简单的业务场景)
	
**方法**
1. 图片压缩
2. css雪碧图
	- 减少你的网站的http请求数量
	* 缺点:整合图片比较大时,一次加载比较慢
3. Image inline
	- 将图片的内容内嵌到html当中减少网站的http请求数量
4. 使用矢量图
	- 使用SVG进行矢量图的绘制,使用iconfont解决icon问题
5. 在安卓下使用webp
 
 **工具**
 
图片格式转换在线网站

# 3.css和js的装载执行的优化
 **概念**
 
 **HTML渲染过程的一些特点**
1. 顺序执行、并发加载
	- 词法分析
	- 并发加载
	- 并发上限(浏览器对请求并发是有上限)
2. 是否阻塞
	- css head中阻塞页面的渲染
	- css阻塞js执行
	- css不阻塞外部脚本的加载
	- js直接引入阻塞页面的渲染
	- js不阻塞资源的加载
	- js顺序执行,阻塞后续js逻辑的执行
3. 依赖关系
	- 页面渲染依赖于css的加载
	- js的执行顺序的依赖关系
	- js逻辑对于dom节点有依赖关系
4. 引入方式
	- 直接引入
	- defer(所有dom已经构建完成后开始加载,不阻塞dom加载,有执行顺序)
	- async(不保证脚本执行顺序,所有脚本不能有依赖性)
	- 异步动态引入js

**优化方法**
1. css样式表置顶(让渲染到位再显示,不影响用户体验)
2. 用link代替import(import缺陷:不支持并发执行,等待页面全部执行完才执行css,产生闪动)(现代版本浏览器两者已经不存在这个缺陷了)
3. js脚本置底(先让css加载)
4. 合理使用js的异步加载能力


# 4.懒加载
 **概念**
 1. 图片进入可视区域之后请求图片资源
 2. 对于电商等图片很多,页面很长的业务场景适用
 3. 减少无效资源的加载
 4. 并发加载的资源过多会阻塞js的加载,影响网站的正常使用

```
<!-- html中 -->
<img src="" class="image-item" lazyload="true" data-original="懒加载后图片的真实地址路径">

<!-- css中 -->
一定要控制每个图片的大小相对一致
<!-- js中 -->
let viewViewHeight=documentElement.clientHeight   //可视区域高度

function lazyLoad(){
	let eles=document.querySelectorAll('img[data-original][lazyload]')  //获取所有懒加载的image元素
	Array.prototype.forEach.call(eles,function(item,index){
		let rect
		if(item.dataset.orginal==='') return
		rect=item.getBoundingClientRect()	//返回元素的大小及其相对于视口的位置
		if(rect.bottom>=0&&rect.top<viewHeight){
			function(){
				let img=new Image()
				img.src=img.dataset.orginal
				img.onload=function(){
					img.src=img.src
				}
				item.removeAttrbute('data-orginal')
				item.removeAttrbute('lazyload')
			}()
		}
	})	
}

lazyload()
document.addEventListener('scroll',lazyload)
```


# 5.预加载
 **概念**
 1. 图片等静态资源在使用之前的提前请求
 2. 资源使用到时能从缓存中加载,提升用户体验
 3. 页面展示的依赖关系维护

**优化方法**
1. image提前加载,通过display:none隐藏
2. 使用image对象
3. 通过XMLHttpRequest对象控制

# 6.重绘与回流
 **概念**
 1. 频繁触发重绘与回流,会导致UI频繁渲染,最终导致js变慢
 2. 频繁触发页面重布局的属性
	- 盒子模型相关属性触发重布局
	- 定位属性及浮动触发重布局
	- 改变节点内部文字结构也会触发重布局
 3. 只触发重绘的属性
 4. 新建dom的过程
	- 获取DOM后分割为多个图层
	- 对每个图层的节点计算样式结果
	- 为每个节点生成图形和位置
	- 将每个节点绘制填充到图层位图中
	- 图层作为纹理上传至GPU
	- 符合多个图层到页面上生成最终屏幕图像
 5. Chrome创建图层的条件(chrome已经相应处理过了)
	- 3D或透视变换css属性
	- 使用加速视频解码的video节点
	- 拥有3D(webGL)上下文或加速的2D是上下文打的canvas
	- 混合插件(Flash)
	- 对自己的opacity做css动画或使用一个webkit变换的元素
	- 拥有加速css过滤器的元素
	- 元素有一个子元素,该子元素在自己的层里
	- 元素有一个z-index较低且包含一个复合层的兄弟元素

**方法**

将频繁重绘回流的DOM元素单独最为一个独立图层,那么这个DOM元素的重绘和回流的影响只会在这个图层中。
1. 避免使用触发重绘/回流的css属性
2. 将重绘/回流的影响范围限制在单独的图层之内


**优化方法**
1. 用translate替代top改变
2. 用opacity替代visibility
3. 不要一条一条的修改dom的样式,预先定义好class,然后修改DOM的className
4. 把DOM离线后修改
5. 不要把dom节点的属性值放在一个循环里当成循环里的变量
6. 不要使用table布局,可能很小的一个小改动会造成整个table的重新布局
7. 动画实现的速度的选择
8. 对于动画新建图层
9. 启用GPU硬件加速


# 7.浏览器存储

 **cookie概念**
 1. http请求无状态,所以需要cookie去维持客户端状态
 2. cookie的生成方式:http response header中的set-cookie.(服务端产生,客户端保存)
 3. cookie的生成方式:js可以通过document.cookie中读写
 4. 使用场景:
	- 用于浏览器端和服务端的交互
	- 客户端自身数据的存储
 5. cookie存储的限制
	- 作为浏览器存储,大小4KB左右
	- 需要设置过期时间expire
 6. 现况:cookie存储数据能力被localstorage替代了
 7. httponly属性实现cookie的禁止读写
 8. 解决cookie中在相关域名下面---cdn的流量损耗方法:cdn的域名和主站的域名要分开

**LocalStorage相关**
 1. HTML5设计出来专门用于浏览器存储的
 2. 大小为5M左右
 3. 仅在客户端使用,不和服务端进行通信
 4. 接口封装较好
 5. 浏览器本地缓存方案

**SessionStorage相关**
 1. 会话级别的浏览器存储
 2. 大小为5M左右
 3. 仅在客户端使用,不和服务端进行通信
 4. 接口封装较好
 5. 对于表单信息的维护

 **IndexedDB**
 1. IndexedDB是一种低级API,用于客户端存储大量结构化数据.
 2. 为应用创建离线版本

**Service Workers相关**
 1. 一个脚本，浏览器独立于当前网页，将其在后台运行，
 2. 具有拦截和处理网络请求的能力，包括以编程方式来管理被缓存的响应
 3. 应用：
	- 使用拦截和处理网络请求的能力，去实现一个离线应用
	- 使用Service Worker在后台运行同时能和页面通信的能力，去实现大规模后台数据的处理

**PWA(progressive Web Apps)相关**
 1. 渐进式的WebApp,通过一系列新的Web特性,配合优秀的UI交互设计,逐步的增强WebApp的用户体验
 2. 可靠:在没有网络的环境中也能提供基本的页面访问,而不会出现"未连接到互联网的页面
 3. 快速:针对网页渲染及网络数据访问有较好的优化
 4. 融入:应用可以被增加到手机桌面,并且和普通应用一样有全屏 推送等特性

# 8.缓存
**缓存相关**
 1. httpheader控制缓存信息
 2. Cache-Control:
	- max-age:缓存的最大时间(从请求时间到max-age这段时间内不去向服务端发请求)(优先级高于Expires)
	- s-maxage:指定public的缓存设备的缓存时间,优先级高于max-age
	- no-cache:发请求到服务端,再判断缓存有没有过期.
	- no-store:完全不对文件使用缓存策略
 3. Expires:缓存过期时间,用来指定资源到期的时间,是服务器端的具体的事件点,告诉浏览器再过期时间前浏览器可以直接从浏览器缓存取数据,而无需再次请求.
 4. Last-Modified/If-Modified-Since:基于客户端和服务端协商的缓存机制
	- last-modefied>response header   if-modified-since>request header 并且需要与cache-control共同使用
	- 缺点:某些服务端不能获取精确的修改时间,文件修改时间改了,但文件内容却没有变
 5. Etag/If_None-Match:文件内容的hash值,通过对比hash值判断文件是否改变
	- etag>response header If-None-Match>request header 并且需要与cache-control共同使用
 6. 分级缓存策略
	- 200状态:当浏览器本地没有缓存或者下一层失效时,或者用户点击了CTL+F5时,浏览器直接去服务器下载最新数据
	- 304状态:这一层由last-modified/etag控制.当一下一层失效时或用户点击refresh,F5时,浏览器就会发送请求给服务器,如果服务端没有变化,返回304给浏览器
	- 200状态(from cache):这一次那个由expires(绝对时间)/cache-control(相对时间)控制,只要没有失效,浏览器只访问自己的缓存
	
# 9.服务端性能优化
 1. Vue渲染面临的问题:框架加载完再渲染页面,会有首屏渲染问题
 2. 在Vue这个层面对性能进行提升:
	- 构建层模板编译
	- 数据无关的prerender的方式
	- 服务端渲染 
