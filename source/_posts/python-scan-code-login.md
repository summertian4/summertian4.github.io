---
title: Python——奇怪的扫码登录
date: 2018-05-20 13:50:46
tags: 
	- Python
	- Django
categories:
	- Python

---

# Python——奇怪的扫码登录

最近在做 Sparrow（还在内测的一个敲好用 Mock 系统😁）的时候遇到了一个需求。Sparrow 服务器是使用 Django 2.0 编写的产品，所以**本文所有的代码背景均为 Django 2.0 环境和 Python 3.6.3 语言，整体是 Vue + Django + SQLite。**

Sparrow 的操作一般都是在网页上操作，而手机客户端往往是用来同步一些简单数据的。那么这里遇到一个和平常 APP 不同的使用场景。

一般来说，一个产品的操作大多是在手机上，那么 PC 客户端和网页版就可以通过已经登录的移动端 APP 扫码登录。

而现在的情况是，Sparrow 的使用大多在网页版，那么，我需要的就是，让移动 APP 用户在网页版已经登录的情况下免去输入用户名、密码的登录操作，让移动 APP 用户扫描网页二维码，完成移动 APP 的登录。

<!-- More -->

## 大致的 Use Case

![](https://raw.githubusercontent.com/summertian4/Images/master/blog/blog_python-scan-code-login-03.png)

##设计思路

### 扫码登录 URL

首先能想到的是，服务器要提供给移动 APP 可以访问的 URL（展现成二维码给 APP 扫描），这个 URL 需要包括

1. user 的唯一标志符
2. 验证码

那么其 URL 的大致模样就是：

```python
frontend/account/quick_login?user_id=<user_id>&verification_code=<verification_code>
```

### 验证码

验证码是从哪里来的？

原因是这样的，如果扫码登录的 URL 永久有效，显然是不合理的，这意味着只要得到了这个 URL，任何人都可以通过这个 URL 随时登录该用户的账号，所以需要有验证码。

同时，**验证码需要附带生成时间**，以此来达**到验证码一分钟有效**的 Feature。为此，设计一个外键为 user 的 Model：

```python
class QuickLoginRecord(models.Model, Dictable):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    create_time = models.DateTimeField(auto_now_add=True)
    update_time = models.DateTimeField(auto_now=True)
    verification_code = models.CharField(max_length=32, null=True, default='')
```

通过 Django 生成的对应数据库为：

![](https://raw.githubusercontent.com/summertian4/Images/master/blog/blog_python-scan-code-login-01.png)

什么时候生成验证码，那当然是生成二维码的时候，所以，这个 URL 不是给移动端请求的，而是给前端来请求的，前端在已登录的情况下，访问该 URL 可以直接传递 user 信息，后端通过拿到 user 信息，生成一条 QuickLoginRecord 记录。

前端访问并拿到验证码的 URL 的大致模样是：

```py
frontend/account/request_quick_login
```

那么整个流程就是（省略了细节处理）：

![](https://raw.githubusercontent.com/summertian4/Images/master/blog/blog_python-scan-code-login-04.png)

## 后端代码

### url.py

```python
urlpatterns = [
	// ···
	path('frontend/account/quick_login', AccountAction.quick_login),
    path('frontend/account/request_quick_login', AccountAction.request_quick_login),
]
```

### models.py

```python
class QuickLoginRecord(models.Model, Dictable):
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    create_time = models.DateTimeField(auto_now_add=True)
    update_time = models.DateTimeField(auto_now=True)
    verification_code = models.CharField(max_length=32, null=True, default='')
```

### account_action.py

这里代码不想看的话，大概描述一下过程：

####  def request_quick_login(request: HttpRequest)

1. 规定 HTTPMethod 必须为 GET 访问
2. 获取 request 中的 user_id 参数
3. 通过 user_id 查询 QuickLoginRecord 记录
4. 如果未查询到结果，新建一个 QuickLoginRecord 记录，设置 user 关联、verification_code（create_time 和 update_time 在 QuickLoginRecordDao 中自动设置）
5. 如果查询到结果，更新 verification_code 字段（ update_time 在 QuickLoginRecordDao 中自动设置）
6. 返回 Success

```python
    @track(AccountRequestQuickLogin)
    def request_quick_login(request: HttpRequest):
        if request.method != CommonData.Method.GET.value:
            return HttpResponse(Response.methodInvalidResponse().toJson(), content_type='application/json')
        user = request.user
        r = QuickLoginRecordDao.get_record_with_user_id(user.id)
        if r is not None:
            r.verification_code = str(uuid.uuid1())
            QuickLoginRecordDao.update_record(r)
            response = Response(Success, 'Success', {'verification_code': r.verification_code})
            return HttpResponse(response.toJson(), content_type='application/json')
        else:
            record = QuickLoginRecord()
            record.user = user
            record.verification_code = str(uuid.uuid1())
            QuickLoginRecordDao.add_record(record)
            response = Response(Success, 'Success', {'verification_code': record.verification_code})
            return HttpResponse(response.toJson(), content_type='application/json')
```

#### def quick_login(request: HttpRequest):

1. 规定 HTTPMethod 必须为 GET 访问
2. 拿到 reqeust 中的 user_id
3. 拿到 reqeust 中的 verification_code
4. 通过 verification_code 获取 QuickLoginRecord 记录
5. 如果记录不存在则表示验证码不存在或过期
6. 如果存在，比较 update_time 字段，判断是否已经超过 60 秒
7. 超过 60 秒返回『验证码过期』
8. 未超过 60 秒，让用户登录
9. 返回 Success

```python
	@track(AccountQuickLogin)
    def quick_login(request: HttpRequest):
        if request.method != CommonData.Method.GET.value:
            return HttpResponse(Response.methodInvalidResponse().toJson(), content_type='application/json')
        user_id = request.GET.get('user_id')
        verification_code = request.GET.get('verification_code')

        record = QuickLoginRecordDao.get_record_with_verification_code(verification_code)
        if record is None:
            response = Response(QuickLoginFailed, '验证码不存在或已过期', {})
            return HttpResponse(response.toJson(), content_type='application/json')

        now = datetime.now(timezone.utc)
        offset = (now - record.update_time).seconds

        if offset > 60:
            response = Response(QuickLoginFailed, '验证码已过期', {})
            return HttpResponse(response.toJson(), content_type='application/json')

        user = AccountDao.get_user_with_id(user_id)
        if user is None:
            response = Response(QuickLoginFailed, '用户不存在', {})
            return HttpResponse(response.toJson(), content_type='application/json')
        user.backend = 'django.contrib.auth.backends.ModelBackend'
        print('用户 ' + user.username + ' 尝试登录')
        auth.login(request, user)
        accountInfo = User.objects.get(id=user.id)
        response = Response(Success, 'Success', {'id': accountInfo.id,
                                                 'username': accountInfo.username,
                                                 'email': accountInfo.email})
        return HttpResponse(response.toJson(), content_type='application/json')
```

#### quick_login_record_dao.py

```python
class QuickLoginRecordDao:
    @staticmethod
    def add_record(record):
        record.save()

    @staticmethod
    def get_record_with_user_id(user_id):
        try:
            record = QuickLoginRecord.objects.get(user_id=user_id)
            return record
        except:
            return None

    @staticmethod
    def update_record(record):
        result = QuickLoginRecord.objects.filter(id=record.id).update(
            verification_code=record.verification_code,
            update_time=datetime.datetime.now())
        if result > 0:
            return True
        else:
            return False

    @staticmethod
    def get_record_with_verification_code(code):
        try:
            record = QuickLoginRecord.objects.get(verification_code=code)
            return record
        except:
            return None
```

## 前端代码

前端的效果是这样的：

![](https://raw.githubusercontent.com/summertian4/Images/master/blog/blog_python-scan-code-login-02.gif)

在已登录的状态下，点击右上角的『客户端扫码登录』按钮，弹出二维码。

动效、模态窗什么的就不过多展示代码了，只关注主流程的代码：

### 导航栏按钮

```html
<p class="nav-item" v-if="account.status">
	<button class="button is-primary" type="submit" @click="openModalImage">客户端扫码登录		</button>
</p>
```

### openModalImage 函数

```js
  openModalImage () {
    const imageModal = openImageModal()
    imageModal.loading = true
    var baseUrl = window.location.protocol + '//' + window.location.host
    request('/frontend/account/request_quick_login', {
      method: 'get'
    }).then((response) => {
      var verificationCode = response.data.verification_code
      var url = baseUrl + '/frontend/account/quick_login' + '?' +
        'verification_code=' + verificationCode + '&' +
      'user_id=' + this.accountInfo.id

      QRCode.toDataURL(url)
        .then(url => {
          imageModal.imgUrl = url
          imageModal.loading = false
          imageModal.$children[0].active()
        })
        .catch(err => {
          console.error(err)
        })
    }).catch((response) => {
      notification.toast({
        message: response.message,
        type: 'danger',
        duration: 2000
      })
    })
  }
```

## iOS代码

iOS 代码就不展示了，就是扫码访问二维码里的 URL，再加上一些非法 URL 的判断即可。

## 总结

过完整个流程后，可以感觉到，类似于支付宝的扫码支付。给出一个定时刷新的二维码，供给客户端进行扫码登录。

当然，还有可以完善的地方，比如前段在打开了二维码模态窗时，每 60 秒进行一次定时刷新。

----

有什么问题都可以在博文后面留言，或者微博上私信我，或者邮件我 <coderfish@163.com>。

博主是 iOS 妹子一枚。

希望大家一起进步。

我的微博：[Lotty小鱼](http://weibo.com/coderfish/)

