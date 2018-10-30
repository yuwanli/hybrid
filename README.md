对于hybrid真正的接触和深入了解，是从来到这家公司开始的，之前虽然接触过，但是那个时候使用的都是各种sdk，面试通过后，在来之前，前端团队负责人就跟我说，来之前可以先自己了解一下hybrid。

>**什么是hybrid开发?**

所谓hybrid，顾名思义就是‘混合模式开发’，简单的解释就是前端h5结合native的原声能力，结合各自的优势和长处，打造一个超h5，类原生的前端应用。

说到h5开发和native开发，这里还是搜集总结了各自的优缺点

>**Native/Web/Hybrid的对比**

![hybrid.png](http://h0.hucdn.com/open/201844/1427ff8851944915_1315x742.png)

从这里可以明显看出来，h5的优势非常明显、无需走发版、无版本问题、一套代码多端运行等。就这几点，对于很多需要快速发展起步的中型企业都是非常的重要，他们需要灵活的发布和调整，不希望被审核所限制，当然人力成本上也希望有所控制。

接触hybrid以后就知道了，以前做的jssdk的开发、小程序的开发，以及后续接触的weex开发，其实都是hybrid的开发，只是结合的形式和开发的方式有所区别罢了，其实具体的核心部分都是一样，都是利用h5与native的通讯，让客户端和前端各司其责，然后有效的、高效率的结合。

>**小程序和weex**

目前正火的小程序，其实处于h5和native中间的一个产物，webview作为渲染容器，结合小程序提供的[高性能的原生组件](https://developers.weixin.qq.com/miniprogram/dev/component/)，以及小程序官方提供的很便捷的开发api，开发者可以很快的开发一个类原生的在微信中使用的‘app’，当然还有不得不提的，小程序的发展迭代速度，社区的维护、文档的更新、健全的发布及版本控制机制，以其很多hybrid开发者梦寐以求的pc端的开发调试工具（还开放了远程调试哦）。

weex就比小程序更加的彻底，全部都会转换成原生(想想都很激动)，前端按照h5开发的页面最终都呈现成原生。三端统一这一个跨时代的技术，一直在跌跌撞撞中缓慢前行，之前有了解过且'hello world'过的有React Native、PhoneGap、Cordova但是都很快就放弃了，因为这些都需要‘多栖程序员’才来驾驭。目前我们的项目中就是存在典型的两种技术栈，h5+hybrid以及weex+hybrid，对于两种技术栈的抉择上，即使weex目前仍存在很多问题，但就weex不会有[‘安卓字体上下居中会偏上’](<https://zhuanlan.zhihu.com/p/25808995>)的问题，我就会优先选它！

>**hybrid的实现方式**

最终希望达到的效果就是js和native能自由通讯，目前主流的两种方式是
* url schema的方式 `hybird://changeTitle?title=修改标题&id=1`
* 典型的‘发布订阅’的模式，通过客户端内置在webview中的bridge对象，这里的bridge对于不同的公司使用会不太一样，我们这边安卓使用的是通用的WebViewJavascriptBridge，ios使用的webkit的WebViewJavascriptBridge，当然还有javascript core等其他的实现方式，这里就不班门弄斧做过多的介绍，他们的处理原理都一样，这里统一称之为bridge。

>**url schema**

对于url schema的这种形式，有点类似jsonp的处理方式，需要与native约定好数据格式，然后做h5的请求的拦截，接着分析处理回掉。比如这里的，会拦截所有hybrid://的前端请求，openURL为前端调用的方法，后面即为处理的参数和回掉id，这种方式一般就把回掉函数挂载在window对象下。

那前端是如何发起请求呢？通常的有window.location.href的形式以及iframe的方式，但是location的形式存在一个问题，多次修改href的值，在Native层只能接收到最后一次请求，前面的请求都会被忽略掉。所用通常会使用iframe的形式。

```
window.callbacks[1] = function() {
  // ...
}
var url = 'hybird://changeTitle?title=修改标题&id=1';
var iframe = document.createElement('iframe');
iframe.style.width = '1px';
iframe.style.height = '1px';
iframe.style.display = 'none';
iframe.src = url;
document.body.appendChild(iframe);
setTimeout(function() {
    iframe.remove();
}, 100);
```

>**bridge**

bridge的使用也是需要与native约定数据格式和函数，数据格式好理解，就类似与后端约定数据格式一样。

```
bridge.send({
    target: 'changeTitle',
    data: {
        title: '修改标题'
    },
},() => {
  // deal in changeTitle callback
})
```

这里会是两个类似`send`函数和`doInFinish`函数。`send`就是js将数据对象通过bridge传给native的过程，安卓的`WebViewJavascriptBridge`对应的是`sendMessage`，ios的`webkit.WebViewJavascriptBridge`对应的是`postMessage`。在send数据之前会生成唯一的id(自增或通过时间戳等方式)作为取回掉函数的钥匙，并将回掉函数挂载在bridge对象下。

```
bridge.send = function (message,callback) {
  message.id = new Date().getTime();
  bridge.callbacks[id] = callback;
  if(isAndroid){
    window.WebViewJavascriptBridge.sendMessage(message);
  } else {
    window.webkit.messageHandlers.WebViewJavascriptBridge.postMessage(message);
  }
};
```

`doInFinish`是native调用js的过程，调用的时候会传入js调用native的时候传入的id，这个时候就去取`bridge.callbacks`下对应id的函数然后执行，这里需要注意的是，为了防止对象冗余，所以在执行后会进行销毁。这里也就说明了我们注册了右上角的分享按钮，分享一次后需重新注册一次，因为调用一次回掉函数就被销毁了。

```
function doInFinish(id,result) {
    const callback = bridge.callbacks[id];
    callback(result);
    delete bridge.callbacks[id];
};
```

以上就是hybird的基本原理了，两种形式在业务中都有使用，url schema用于‘声明式‘调用，如打开带有特殊属性(禁用下拉刷新、隐藏分享按钮)的webview容器，就可以在链接上带上固定的参数，如`https://xxxxx/index.html?hybrid_info={'disableRefresh': true,'hideShare': true}`。bridge这种’函数式‘调用，适合在代码中使用，底层做好封装，js就可以执行函数那样去调用native的函数了，这块代码可以参考[@fe-base/hybrid的源码](http://git.husor.com/fe-base/hybrid/tree/master/src)。

>**令人困惑的config注册过程**

在了解基础原理以后，使用函数式调用使用hybrid的时候，直接使用即可，但是看了源码会发现，这里有一个约定好的`config`操作，这里称之为注册的过程。就拿`changeTitle`为例，按照上面的原理分析，按照bridge里分析的直接`send`就可以了，但是实际上是先`config`，之后在进行`changeTitle`的调用。

```
// 具体可参考源码，这里抽象逻辑结构
bridge.send({
    target: 'config',
    data: {
        api: 'changeTitle'
    },
},() => {
  bridge.send({
    target: 'changeTitle',
    data: {
        title: '修改标题'
    },
  },() => {
    // deal in changeTitle callback
  })
})
```

这里也是纠结了一下，觉得注册过程略显冗余，经过分析学习和探讨，这块之所以会有注册的策略，主要的原因有以下几点

1. 防止调用的hybrid方法不存在，js调用之后处于’黑盒’（无法跟踪定位问题）状态，注册之后，调用出错至少能排除’native还没有该方法’这种情况。由于h5的灵活和跨平台性，无法确定和限制h5最终运行的环境，而且很多公司都是有很多app，native层之间的差异性会导致js的环境不可控。
2. 对于后续有可能接入第三方的应用，类似微信的jssdk，不同的主体对于微信的native开放出来的sdk使用权限不一样，通过用appid注册的形式，来确定sdk的使用权限。

这里就出现了一个问题，hybrid每次调用都要注册么，这样岂不是很冗余而且影响性能么？微信的jssdk是下面使用的

```
wx.config({
    debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
    appId: '', // 必填，公众号的唯一标识
    timestamp: , // 必填，生成签名的时间戳
    nonceStr: '', // 必填，生成签名的随机串
    signature: '',// 必填，签名
    jsApiList: ['chooseImage'] // 必填，需要使用的JS接口列表
});

wx.chooseImage({
    count: 1, // 默认9
    sizeType: ['original', 'compressed'], // 可以指定是原图还是压缩图，默认二者都有
    sourceType: ['album', 'camera'], // 可以指定来源是相册还是相机，默认二者都有
    success: function (res) {
        var localIds = res.localIds; // 返回选定照片的本地ID列表，localId可以作为img标签的src属性显示图片
    }
});
```

会发现也是会分两个步骤，先是进行config，成功之后就可以直接调用使用了。那我们的自己的hybrid的原理其实也是一样，只是目前是公司内部使用不存在所谓的appId和signature等，我们的基础使用方法也类似`hybrid('changeTitle').changeTitle({title: '修改标题'},() => {})`,其中同样也是包含了两个过程，先注册然后进行调用。这里要说一下之所以能这样链式调用，是因为这里注册完以后，会以此为函数名挂载在hybrid对象下，且返回当前对象。

```
let a = {}
a = (name) => {
  // ... config finish
  a[name] = (params) => {
    //deal with params
    console.log(params)
  }
  return this //需返回当前对象
}
a('b').b('test')  // 'test'
```
