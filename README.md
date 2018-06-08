# geetest-ruby-sdk

这个是3.0版的SDK.

可以直接拿来使用.

对于不太明白的同学,可以安装好java版之后,看例子. [java版本demo](https://docs.geetest.com/install/deploy/server/java)

下面是我摘录出来的过程,希望对大家有用.

## 工作过程

### 第一步, 定义HTML 页面

先定义一个登陆页面（ 注意定义的 submit1,   jquery , gt.js ）

```
<form action="gt/ajax-validate1" method="post">
    <h2>大图点击Demo，使用表单进行二次验证</h2>
    <br>
    <div>
        <label for="username1">用户名：</label>
        <input class="inp" id="username1" type="text" value="极验验证">
    </div>
    <br>
    <div>
        <label for="password1">密码：</label>
        <input class="inp" id="password1" type="password" value="123456">
    </div>
    <br>
    <div>
        <label>完成验证：</label>
        <div id="captcha1">
            <p id="wait1" class="show">正在加载验证码......</p>
        </div>
    </div>
    <br>
    <p id="notice1" class="hide">请先完成验证</p>
    <input class="btn" id="submit1" type="submit" value="提交">
</form>


<!-- 注意，验证码本身是不需要 jquery 库，此处使用 jquery 仅为了在 demo 使用，减少代码量 -->
<script src="http://apps.bdimg.com/libs/jquery/1.9.1/jquery.js"></script>

<!-- 引入 gt.js，既可以使用其中提供的 initGeetest 初始化函数 -->
<script src="gt.js"></script>

<script>
    var handler1 = function (captchaObj) {
        $("#submit1").click(function (e) {
            var result = captchaObj.getValidate();
            console.log("== result: " + result);
            if (!result) {
								alert('请进行人机验证!')
                e.preventDefault();
            }
        });
        // 将验证码加到id为captcha的元素里，同时会有三个input的值用于表单提交
        captchaObj.appendTo("#captcha1");
        captchaObj.onReady(function () {
            $("#wait1").hide();
        });
        // 更多接口参考：http://www.geetest.com/install/sections/idx-client-sdk.html
    };
    $.ajax({
        url: "/gee_test_register?t=" + (new Date()).getTime(), // 加随机数防止缓存
        type: "get",
        dataType: "json",
        success: function (data) {
            // 调用 initGeetest 初始化参数
            // 参数1：配置参数
            // 参数2：回调，回调的第一个参数验证码对象，之后可以使用它调用相应的接口
            initGeetest({
                gt: data.gt,
                challenge: data.challenge,
                new_captcha: data.new_captcha, // 用于宕机时表示是新验证码的宕机
                offline: !data.success, // 表示用户后台检测极验服务器是否宕机，一般不需要关注
                product: "float", // 产品形式，包括：float，popup
                width: "100%"
                // 更多配置参数请参见：http://www.geetest.com/install/sections/idx-client-sdk.html#config
            }, handler1);
        }
    });
</script>

```

上面的代码, 可以用最普通的html浏览器打开.

### 第二步. 该页面会加载 动态验证码

请注意上面的 页面加载后， 会自动加载：   `/gee_test_register?t=1528436582` 这个url(t= 后面的参数是当前时刻,可以认为是随便写的参数),
这个就是我们后端要做的事儿。

后端代码：


```
# 记得务必引入这个 .rb 文件,对于sinatra放在项目根目录就好. 对于Rails建议放在lib目录下,
# 总之可以被加载就行.
require './geetest_sdk'

# 这个是sinatra的代码. rails一样的. 相信大家对Rails更熟悉.
get "/gee_test_register" do

  # 这个sdk 的两个参数: 1.  id   2. key .都是在Geetest后台申请的
	sdk = GeetestSdk.new config[:gee_test_id], config[:gee_test_key]

	result_from_gee_test_server = sdk.pre_process('随便用个字符串')

	# 返回 向服务器申请得到的 流水号
	json result_from_gee_test_server
end

```


会根据这个，返回类似于这样的结果的response:

```
{
	// 表示申请流水号成功
  success: 1,

	// 这个 challenge 是流水号， 从 gee test服务器返回的。
  challenge: "9aa704bb66dd93590adb5cdb3536db92",

	// 这个是我们申请的 app id,
  gt: "433133f5197e42bf5748be2230dbffd4"
}
```

(这个随意)同时，服务器端要在session中保存 “gee_test_server_status ” 和 "user_id" 的值

- user_id 就是当前服务器的一个标志。 是可选参数。 似乎是可以被GeeTest用作分析和统计
- gee_test_server_status 则是看Geetest服务器是否可以正常工作. 对于新手,墙裂建议不必考虑这里. 先把代码跑起来才是真的.

### 第三步 WEB页面加载好一系列的js, css ,显示人机验证弹窗

接下来， WEB页面就会自动加载一系列的东东。 显示拖拽窗口啥的。

当拖拽完毕后， js端会自动把结果保存到一个对象： `captchaObject`.

（这个是前端的对象）允许用户点击表单的“下一步”按钮。（例如 登陆，注册，发送验证码 等等）

所以,这个时候,我们可以对WEB页面的 form 提交,做个判断, 起作用的就是下面的代码:

```
$("#submit1").click(function (e) {
		var result = captchaObj.getValidate();
		console.log("== result: " + result);
		if (!result) {
				alert('请进行人机验证!')
				e.preventDefault();
		}
});

```

4. 然后， url 被 我们的后端处理。  我们的后端的代码中， 看起来是这样的：

ruby 代码:

```
# 在原来的验证表单的Action前面,加上对于 验证码的判断即可:

# sinatra 风格. 处理  /login 这个请求
post "/login" do

	# 这里是新增的代码. 开始

  # 新搞起一个sdk.  这个sdk 的两个参数: 1.  id   2. key .都是在Geetest后台申请的
	sdk = GeetestSdk.new config[:gee_test_id], config[:gee_test_key]

	# 根据表单提交时自动传递进来的3个参数进行判断.
	result = sdk.success_validate params[:geetest_challenge], params[:geetest_validate], params[:geetest_seccode]

  # 如果 人机验证不通过的话, 跳转回 登陆页面,给出500错误. (出错信息大家自己填)
	unless result
		status 500
		return render @template_engine, :login
	end
	# 这里是新增的代码. 结束

	# 下面是原来的正常代码.
  ......
end
```



## 工作原理

虽然网上有一些文章, 是专门破解 Geetest的, 但是都是针对Geetest  2.0的部分.

对于Geetest 3.0的部分, 目前来看没有办法. 因为3.0是一个三方认证(共识) -_-!

具体看 [通讯流程](https://docs.geetest.com/install/overview/flowchart)

可以看出,如果浏览器希望绕过 应用服务器, 直接与 Geetest服务器做沟通的话,是不可能的.




