## COS对比技术报告

本报告将从`api`角度分析，比较我们云存储产品`cos`和友商云存储产品（阿里云`oss`和亚马逊`S3`）之间的区别。由于`S3`和`oss`提供的`api`以`xml api`为主，因此这里侧重于`xml api`的比较，结合部分控制台请求抓包，尽可能做到有理有据。

## 提纲：
* 公共头部
* 鉴权部分（签名）
* GetService
* Bucket api
    * 删除 bucket
    * bucket 访问日志
    * bucket 跨域设置
    * bucket 访问权限
    * bucket 防盗链
    * bucket 静态网站托管
    * bucket 生命周期
    * 创建 bucket
    * 获取 bucket 内的 Object 列表
    * 获取 bucket 具体信息
    * S3 特有的 bucket api 介绍
* Object api
    * Object 单次删除
    * Object 批量删除
    * 获取 Object
    * Object 访问权限
    * Object 上传（非分块）
    * Object 追加上传
    * Object 分块上传
        * 初始化分块上传
        * 分块上传
        * 分块上传完成
        * 分块上传任务抛弃
        * 分块上传列表查询
        * 当前分块上传任务列表查询（非必须）
    * Object 拷贝

## api 情况概览

|api 类型|腾讯云 cos|阿里云 oss|亚马逊 S3|
|:--|:--|:--|:--|
|GetService|不支持|支持|支持|
|bucket 删除 |支持|支持|支持|
|bucket 访问日志|不支持|支持获取、关闭和设置|支持获取|
|bucket CORS跨域设置|不支持|支持获取、关闭和设置|支持获取、关闭和设置|
|bucket 访问权限|支持匿名访问权限和用户级别访问权限|仅支持匿名访问权限|支持匿名访问权限和用户级别访问权限|
|bucket 防盗链设置|不支持|支持获取和设置|支持获取和设置|
|bucket 静态网站托管|不支持|支持获取、关闭和设置|支持获取、关闭和设置|
|bucket 生命周期|不支持|支持获取、关闭和设置|支持获取、关闭和设置|
|bucket 创建|支持|支持|支持|
|获取 bucket 内 Object 列表|支持|支持|支持|
|获取特定 bucket 具体信息|不支持|支持|不支持，但是可以通过GetService实现|
|bucket 事件通知|不支持|不支持|支持|
|bucket 标签|不支持|支持|支持|
|bucket 跨区域复制|不支持|不支持|支持|
|bucket 策略|不支持|不支持|支持|
|bucket 版本控制|不支持|不支持|支持|
|Object 单次删除|支持|支持|支持|
|Object 批量删除|不支持|支持|支持|
|Object 获取 |支持|支持|支持|
|Object 访问权限|支持获取和设置|支持获取和设置|支持获取和设置|
|Object 上传（非分片）|支持，有 PUT 和 POST 两种请求|支持，有 PUT 和 POST 两种请求|支持，有 PUT 和 POST 两种请求|
|Object 追加上传|支持|支持|不支持|
|Object 初始化分块上传|支持|支持|支持|
|Object 分块上传|支持|支持|支持|
|Object 完成分块上传|支持|支持|支持|
|Object 分块上传任务抛弃|支持|支持|支持|
|Object 分块上传列表查询|支持|支持|支持|
|当前分块上传任务列表|不支持|支持|不支持|
|文件拷贝|部分支持，可以通过移动文件JSON api实现|支持，对大文件和小文件分别处理|支持，对大文件和小文件分别处理|



## 一、公共头部

### cos 公共请求头部（如图1.1.1），公共响应头部（如图1.1.2）  

![图1.1.1][1]

                                                图1.1.1

![图1.1.2][2]

                                                图1.1.2
                                                

### oss 公共请求头部（如图1.2.1），公共响应头部（如图1.2.2）  

![图1.2.1][3]

                                                图1.2.1
                                                                                              
![图1.2.2][4]

                                                图1.2.2

### S3 公共请求头部（如图1.3.1），公共响应头部（如图1.3.2）

![图1.3.1(1)][5]
![图1.3.1(2)][6]

                                                图1.3.1
                                                
![图1.3.2(1)][7]
![图1.3.2(2)][8]

                                                图1.3.2

### 小结：
`oss`的公共头部简单，相当于`cos`和`S3`的公共部分，无需特别讨论，具体分析`cos`和`S3`的区别。`S3`公共请求头部的抓包（如图1.3.3），公共返回头部的抓包（如图1.3.4），可以看出，`S3`控制台请求进行了一次转发，将真正的请求头部写在`request payload`中。公共请求头部方面，`cos`和`S3`存在以下几点不同：
* 公共请求头部方面
    * `cos`使用`x-cos-content-sha1`字段，用于校验文件是否发生变化，具体方法是将文件进行`SHA1`编码，生成`160-bit`内容校验值。`S3`使用`x-amz-content-sha256`字段检验文件，使用`sha256`对文件进行校验。
    * `S3`额外提供了一个字段记录当前发起者的时间信息，`x-amz-date`，但是其实`s3`提供了`Date`字段做相同的事情（`cos`也有），因此这个字段意义不明，文档里也表示如果指定了`Authorization`，那么必须在`Date` 或者`x-amz-date`选择填一个。
    * `S3`提供了一个特殊字段`x-amz-security-token`，用于两个场景，一个是使用亚马逊支付`Amazon DevPay`，另一个是在用户使用的是临时安全证书的时候。
* 公共响应头部方面
    * `S3`提供`x-amz-delete-marker`字段，返回该对象是不是一个删除标记。
    * `S3`的`x-amz-request-id`和`x-amz-id-2`和`cos`的`x-cos-request-id`与`x-cos-trace-id`一致，前者相当于问题的编号，后者是用户出错的具体信息。
    * `S3`提供的`x-amz-version-id`字段，返回该对象的版本号，`S3`上传的对象存在版本控制，可以记录同一个文件的不同版本，比如我上传A文件，然后本地修改了，再次上传在同一个文件目录中，这时就会有A文件的两个版本。


![图1.3.3][9]

                                                图1.3.3
 
![图1.3.4][10]

                                                图1.3.4                             

## 二、鉴权部分（签名）

### cos签名可以使用两种形式添加在 HTTP请求中，分别是：
* `HTTP Authorization Header`：除了`POST`操作以外，其他的签名都可以通过HTTP请求头部中的 `Authorization`字段传入。
* `URL参数`：在发起`POST`请求时，需要以`URL`后携带参数的方式传入签名。

### HTTP头部传入
Authorization 需要传入的信息字段（如图2.1.1）
#### 签名算法流程
* `SignKey`：携带有效时间，并通过`SecretKey`进行`HMAC-SHA1`加密的密钥串。
* `FormatString`：将请求经过一定规范格式化后的字串。
* `StringToSign`：包含校验算法、请求有效时间和`Hash`校验后的`FormatString`的字符串。
* `Signature`：加密后的签名，使用`SignKey`与`StringToSign`通过`HMAC-SHA1`加密的字符串，填入`q-signature`。

![图2.1.1][11]

                                                图2.1.1

### URL参数传入
#### 签名分为多次有效签名和单次有效签名两种（如图2.1.2）
* 多次有效签名：签名中部分不绑定文件`fileid`，部分绑定文件`fileid`，开启`token防盗链`的下载、文件简单上传、文件分片上传可以绑定`fileid`，同时支持前缀匹配。多次有效签名需要设置一个大于当前时间的有效期，有效期内此签名可多次使用，有效期最长可设置三个月。
* 单次有效签名：签名中绑定文件`fileid`，有效期必须设置为0，此签名只可使用一次，且只能应用于被绑定的文件或文件夹操作。

#### 签名算法流程
* 拼接明文字符串`Original`，明文字符串`Original`按照签名的类型可划分为“多次”和“单次”有效签名。具体需要拼接的参数（如图2.1.3）
* 将明文字符串转化为签名，拼接好签名的明文字符串`Original`后，用已经获取的`SecretKey`对明文串进行 `HMAC-SHA1`加密，得到`SignTmp`。`SignTmp = HMAC-SHA1(SecretKey, Original)`
* 将密文串`SignTmp`放在明文串`Origin`前面，拼接后进行`Base64Encode`算法，得到最终的签名`Sign`。`Sign = Base64 (append(SignTmp, Original))`

![图2.1.2][12]

                                                图2.1.2

![图2.1.3][13]

                                                图2.1.3

### oss 鉴权和cos 类似，这里只归纳不同点：
* `Authorization`传入签名算法流程不同（如图2.2.1）
* `URL`参数传入不同
    * `oss`除了需要在`query`中传入签名`Signature`之外，还需要`AccessKeyId`（`cos`的`secretId`）和`Expires`（签名过期时间）。`cos`中只需要传入签名，因为`Expires`已经在计算签名时算入了。


![图2.2.1][14]

                                                图2.2.1

### s3 鉴权和 cos 类似，不同点在于：
* `Authorization`传入签名，需要传入的信息不同（如图2.3.1）
* `URL`参数传入（如图2.3.2），具体签名计算流程不同（如图2.3.3）

![图2.3.2][15]

                                                图2.3.2
                                                
![图2.3.3][16]

                                                图2.3.3
                                              
### 小结：
##### 此处着重于`S3`和`cos`的鉴权对比，具体分析`cos`和`S3`的鉴权区别：
* 在进行`URL`传入时，除了签名，还需要传入额外字段数据，如：
    * `X-Amz-Algorithm`（加密算法）
    * `X-Amz-Credential`（请求凭据，包括`acces-key`，`date`，`aws-region`和`aws-service`）
    * `X-Amz-Date`（时间戳，格式为yyyyMMddTHHmmssZ ）
    * `X-Amz-Expires`（过期时长，和`cos`的过期时间点不同）
    * `X-Amz-SignedHeaders`（要用于计算签名的请求头部，包括`x-amz-*`字段和`HTTP host`）
* 签名算法不同，如：
    * `S3`进行格式化请求时候，需要对`resource`进行`UriEncode`，对每一`query`字段的`key`和`value`都要进`UriEncode`，对每一个请求头部的`key`进行`lowercase`，对头部的`value`进行`Trim`处理，还需要拼接每一个求头部的`key`值。
    * `S3`进行`StringToSign`时，需要拼接加密类型（`AWS4-HMAC-SHA256`），时间戳，范围（`scope`）`<yyyymmdd>/<aws region >/s3/aws4_request`，以及`Hex`处理过的`SHA256Hash`请求字符串。
    * `S3`需要算出一个`signKey`，要使用到时间戳，`aws region`，`aws service`进行多次`HMAC-SHA256` 计算。
    * `S3`利用`signKey`和`StringToSign`进行最后一次`HMAC-SHA256`计算，并且进行`Hex`编码。

## 三、Get Service

### cos 没有提供获取用户所有 bucket 信息的 api ，暂时不提供 GetService 服务

### oss 提供 Get Service api，用于获取用户拥有的所有bucket信息
* 对于服务地址作`Get`请求可以返回请求者拥有的所有`Bucket`，其中“/”表示根目录。
* 请求参数（如图3.2.1），响应元素（如图3.2.2）

![图3.2.1][17]

                                                图3.2.1
           
![图3.2.2(1)][18]
![图3.2.2(2)][19]

                                                图3.2.2


### S3 也提供 Get Service api
* 对于`GetService`只需要提供`公共请求头部`即可，无需提供额外的请求元素。
* 响应元素（如图3.3.1），和`oss`的类似。

![图3.3.1][20]

                                                图3.3.1
                                               
### 小结：
#### oss 的GetService 和 S3 的类似，唯一的区别是：
* `oss`可以选择进行筛选，可以发送额外请求元素，如：     
    * `prefix`（特定的`bucket前缀`）
    * `marker`（设定结果从`marker`之后按`字母排序`的第一个开始返回）
    * `max-keys`（限定此次`bucket`返回的`最大数`）。


## 四、删除 bucket
### cos 提供删除 bucket 的 api
* 需要事先清空`bucket`内的所有`文件/文件夹`或者`部分上传成功`的文件标记
* 请求语法（如图4.1.1）

![图4.1.1][21]

                                                图4.1.1
                                               
### oss提供删除 bucket 的 api
* 需要事先清空`bucket`内的所有`文件/文件夹`和`文件碎片`（分片上传失败产生，等于`cos部分上传成功`文件）
* 请求删除的`bucket`不存在、不为空或者操作者权限不足，均会报错  
* 请求语法（如图4.2.1），响应报文（如图4.2.2）

![图4.2.1][22]

                                                图4.2.1

![图4.2.2][23]

                                                图4.2.2
                                                
### S3 提供删除 bucket 的 api
* 需要事先清空`bucket`内的所有`文件/文件夹`和`文件碎片`
* 请求语法（如图4.3.1），响应报文（如图4.3.2）

![图4.3.1][24]

                                                图4.3.1
                                                
![图4.3.2][25]

                                                图4.3.2
                                                
### 小结：
#### 三者提供的删除bucket api 基本一致，这里提供控制台抓包数据对比（仅显示请求部分），cos 请求（如图4.1.2），oss 请求（如图4.2.3），S3请求（如图4.3.3）。 可以看出三者最大的区别在于：
* `cos`在删除时需要额外提供`维度`相关信息，如：     
    * `appid`
    * `projectId`
* `oss `在删除时需要额外提供`权限认证`相关信息，如：     
    * `token`字段
    * 删除短信验证码确认
* `S3`在删除时需要额外提供`删除限制`相关信息，如：     
    * `checkBucketUnderLimit`字段
    * `shouldDeleteBucket`字段
    * `maxItemDeleteLimit`字段，由于在控制台删除非空`bucket`成功，因此该字段应当是用于删除非空`bucket`时，`bucket`内最大的`Object`数量，如果超出，则删除失败。
 
![图4.1.2][26]

                                                图4.1.2

![图4.2.3][27]

                                                图4.2.3
 
![图4.3.3][28]

                                                图4.3.3

## 五、bucket 访问日志

### cos没有访问日志功能，控制台和api均不提供

### oss 提供访问日志功能，具体提供的 api 有 
* 获取`日志设置`请求（如图5.2.1）
* 获取`日志设置`响应元素（如图5.2.2）
* 关闭`日志设置`请求（如图5.2.3）
* 获取`日志`请求（如图5.2.4），请求元素（如图5.2.5）

![clipboard.png](/img/bVHAnE)

                                                图5.2.1


![clipboard.png](/img/bVHAnS)

                                                图5.2.2

![clipboard.png](/img/bVHAoC)

                                                图5.2.3

![clipboard.png](/img/bVHAoE)

                                                图5.2.4

![clipboard.png](/img/bVHAoL)

                                                图5.2.5

### S3 提供访问日志功能，具体提供的 api 有 
* 获取`日志`请求（如图5.3.1），请求元素（如图5.3.2）


![clipboard.png](/img/bVHArE)

                                                图5.3.1
                                                
![clipboard.png](/img/bVHArP)

                                                图5.3.2

### 小结：
* `oss`和`S3`在访问日志获取方面近似
* `oss`额外提供`访问日志设置`api，`S3`不提供设置`api`，但是可以在`S3控制台`中设置

## 六、bucket 跨域设置

### cos 不提供设置bucket属性的api
* 只能在`cos控制台`的`基础配置`中进行跨域访问`CORS`设置。
* `cos控制台`进行`CORS`设置的请求抓包（如图6.1.1）。

![clipboard.png](/img/bVHAtu)

                                                图6.1.1

### oss 提供设置 bucket CORS 属性的api
* 可以在控制台进行相应的设置,`oss控制台`进行`CORS`设置的请求抓包（如图6.2.1）
* `CORS设置`api请求（如图6.2.2）、请求元素（如图6.2.5）
* `CORS关闭`api请求（如图6.2.3）
* `CORS获取`api请求（如图6.2.4）

![clipboard.png](/img/bVHAvl)

                                                图6.2.1

![clipboard.png](/img/bVHAvy)

                                                图6.2.2

![clipboard.png](/img/bVHAvI)

                                                图6.2.3

![clipboard.png](/img/bVHAvT)

                                                图6.2.4

![clipboard.png](/img/bVHAv1)

                                                图6.2.5

### S3 提供设置 bucket CORS 属性的api
* 可以在控制台进行相应的设置,`S3控制台`进行`CORS`设置的请求抓包（如图6.3.1），设置的CORS策略（如图6.3.2）
* `CORS设置`api请求（如图6.3.3）
* `CORS关闭`api请求和`oss`类似
* `CORS获取`api请求和`oss`类似


![clipboard.png](/img/bVHAzu)

                                                图6.3.1

![clipboard.png](/img/bVHAzX)

                                                图6.3.2

![clipboard.png](/img/bVHAzM)

                                                图6.3.3

### 小结：
* 在`CORS`设置方面，`S3`和`oss`提供的api更加全面。
* `oss`和`cos`控制台设置`CORS`请求以`FormData`形式传递参数，而`S3`主要以`xml`类型的`CORS配置文本`进行参数传递，而非使用`FormData`。

## 七、bucket 访问权限

### cos 提供访问权限设置的api
* 可以在控制台设置`bucket`的访问权限
* 两种权限选择：`公有读私有写`、`私有读写`
* `Bucket`访问权限设置`api`，请求语法（如图7.1.1），头部信息（如图7.1.2），请求内容（如图7.1.3）
* 控制台`bucket`访问权限设置请求抓包（如图7.1.4）



![clipboard.png](/img/bVHAD2)

                                                图7.1.1

![clipboard.png](/img/bVHAEa)

                                                图7.1.2

![clipboard.png](/img/bVHAEe)

                                                图7.1.3


![clipboard.png](/img/bVHADV)

                                                图7.1.4

### oss 提供访问权限设置的api
* 可以在控制台设置`bucket`的访问权限
* 三种权限选择：`公有读私有写`、`私有读写`和`公有读写`
* `bucket`访问权限设置`api`和`cos`一致
* 控制台`bucket`访问权限设置请求也和`cos`的一致

### S3 提供访问权限设置的api
* 可以在控制台设置`bucket`的访问权限，控制台请求抓包（如图7.3.1）
* 三种权限选择：简单读写权限（读写的公私有，和oss一致），特定授权者（授权者身份和授予的权限）
* `bucket`访问权限获取`api`，请求语法（如图7.3.2）
* `bucket`访问权限设置`api`，请求语法（如图7.3.3）


![clipboard.png](/img/bVHAIl)

                                                图7.3.1

![clipboard.png](/img/bVHAHm)

                                                图7.3.2

![clipboard.png](/img/bVHAHC)

                                                图7.3.3
### 小结：
* `cos`和`oss`的访问权限比较类似，都是设定一些简易的访问策略（`读写的公有或私有`）
* `S3`的访问权限更加细致，对比`cos`和`S3`的抓包信息以及api可以看出，`S3`可以精确到被授权者身份（例如其他`AWS`用户）及其被授予的权限类型（`上传权限`，`查看权限`，`编辑权限`），授权机制更加严格。
* 新版`cos`的访问权限设置api也提供针对`用户级别`（`uin`）的授权，可以授予特定用户读写权限，但是S3将写权限进一步分为`上传权限`和`编辑权限`，在某些特定的使用场景下，如开放公共空间供其他用户上传文件，但是不希望删除和修改文件属性，就可以只开放给特定用户上传权限，这方面`S3`处理的更细致。



## 八、bucket 防盗链

### cos 不提供 bucket 防盗链设置的 api
* 只能在`cos控制台`的`基础配置`中进行`防盗链`设置。
* `cos控制台`进行`防盗链`设置的请求抓包（如图8.1.1）。


![clipboard.png](/img/bVHAOS)

                                                图8.1.1

### oss 提供 bucket 防盗链设置的 api
* 可以在`oss控制台`进行相应的设置，请求和`cos`基本一致（如图8.2.1）
* `api`方面，防盗链`设置`请求（如图8.2.2）
* `api`方面，防盗链`获取`请求（如图8.2.3）


![clipboard.png](/img/bVHAWQ)

                                                图8.2.1

![clipboard.png](/img/bVHAQP)


                                                图8.2.2


![clipboard.png](/img/bVHAQW)

                                                图8.2.3
                                                

### S3 提供 bucket 防盗链设置的 api
* `S3`提供的防盗链功能比较隐蔽，它是作为`S3 bucket策略`的一部分存在的
* S3 的`bucket 策略`包括如下几种使用场景
    * 在添加条件的情况下向多个账户授予权限
    * 向匿名用户授予只读权限
    * 限制对特定 IP 地址的访问权限
    * 允许 IPv4 和 IPv6 地址
    * 限制对特定 HTTP 引用站点的访问 （防盗链）
    * 向 Amazon CloudFront Origin Identity 授予权限
    * 添加存储桶策略以要求进行 MFA 身份验证
    * 在授予上传对象的交叉账户许可的同时，确保存储桶拥有者拥有完全控制
    * 向 Amazon S3 清单和 Amazon S3 分析功能授予权限
* 具体的其他功能和使用可以通过如下链接访问：
https://docs.aws.amazon.com/zh_cn/AmazonS3/latest/dev/example-bucket-policies.html
* 这里我们单独讨论起防盗链功能。默认情况下，所有`S3`资源都是`私有的`，因此只有创建资源的`AWS 账户`才能访问它们。要允许从您的网站对这些对象进行读取访问，您可以添加一个`存储桶策略`允许`s3:GetObject`权限，并附带使用`aws:referer`键的条件，即获取请求必须来自特定的网页。以下策略指定带有`aws:Referer`条件键的`StringLike`条件（如图8.3.1）。


![clipboard.png](/img/bVHAUP)

                                                图8.3.1
 
### 小结：
* 在控制台设置防盗链设置方面，`cos`和`oss`基本一致。
* `S3`的防盗链设置大有不同，比如说`S3`防盗链是涉及bucket内的`S3:GetObject`操作，需要按照控制台的`策略生成器`生成对应的`JSON document`，然后才能使用，我创建了一个`GetObject`的防盗链设置（如图8.3.2），发起防盗链设置请求抓包（如图8.3.3），可以看出防盗链设置是bucket policy 的一部分。
* `S3`相当于将`访问`相关的部分都综合到一起，而`cos`和`oss`更加倾向于分开，因此`cos`和`oss`在功能区分方面更清晰，使用起来更方便。


![clipboard.png](/img/bVHAYZ)

                                                图8.3.2

![clipboard.png](/img/bVHAY4)

                                                图8.3.3


## 九、bucket 静态网站托管

### cos 不提供 bucket 静态网站托管的 api
* 只能在`cos控制台`中进行`静态网站`设置。
* `cos控制台`进行`静态网站`设置的请求抓包（如图9.1.1）。


![clipboard.png](/img/bVHAZK)

                                                图9.1.1

### oss 提供 bucket 静态网站相关的 api
* 静态网站设置请求（如图9.2.1）。
* 静态网站获取请求（如图9.2.2）。
* 静态网站关闭请求（如图9.2.3）。
* 控制台请求（如图9.2.4）


![clipboard.png](/img/bVHAZR)

                                                图9.2.1

![clipboard.png](/img/bVHAZT)

                                                图9.2.2

![clipboard.png](/img/bVHAZU)

                                                图9.2.3

![clipboard.png](/img/bVHA0h)

                                                图9.2.4


### S3 提供 bucket 静态网站相关的 api
* 静态网站设置请求和`oss`的一致。
* 静态网站获取请求和`oss`的一致。
* 静态网站关闭请求和`oss`的一致。
* 控制台请求（如图9.3.1）

![clipboard.png](/img/bVHA0t)

                                                图9.3.1

### 小结：
* 在控制台设置静态网站方面，`cos`和`oss`基本一致，请求类似，请求参数都是`FormData`形式传输。
* 控制台方面，`S3`请求参数依然使用xml文本传输。

## 十、bucket 生命周期
### cos 不提供 bucket 生命周期设置

### oss 提供 bucket 生命周期相关的 api
* 生命周期设置请求（如图10.2.1）。
* 生命周期获取请求（如图10.2.2）。
* 生命周期关闭请求（如图10.2.3）。


![clipboard.png](/img/bVHBnc)

                                                图10.2.1

![clipboard.png](/img/bVHBnu)

                                                图10.2.2
  
![clipboard.png](/img/bVHBnx)

                                                图10.2.3


### S3 提供 bucket 生命周期相关的 api
* 生命周期设置请求和`oss`类似，但是除了可以设置`删除`规则，还可以设置`存储迁移`规则（存储类型变化，如图10.3.1）
* 生命周期获取请求`oss`类似。
* 生命周期关闭请求`oss`类似。 


![clipboard.png](/img/bVHBzo)

                                                图10.3.1

### 小结：
* 在生命周期方面，`oss`和`S3`基本一致，但是值得注意的是`oss`只能设置`Object`或者`Object碎片`的`删除`规则，而`S3`除了设置`删除`规则，还可以设置`存储类型迁移`规则，如`标准存储`迁移至`低频存储`和`Glacier 存储类`（如图10.3.2）


![clipboard.png](/img/bVHBDL)

                                                图10.3.2



## 十一、创建 bucket
### cos 提供创建 bucket 的 api
* 创建bucket请求语法（如图11.1.1）。
* 可以设置一些与权限相关的头部（如图11.1.2）。


![clipboard.png](/img/bVHBGY)

                                                图11.1.1

![clipboard.png](/img/bVHBG4)

                                                图11.1.2

### oss 提供创建 bucket 的 api
* 创建bucket请求语法（如图11.2.1）。
* 可以进行简易的访问权限设置，如`x-oss-acl`设置匿名读写的公私有

![clipboard.png](/img/bVHBHY)

                                                图11.2.1

### S3 提供创建 bucket 的 api
* 创建bucket请求语法（如图11.3.1）。
* 可以设置一些与权限相关的头部（如图11.3.2）。


![clipboard.png](/img/bVHBIs)

                                                图11.3.1

![clipboard.png](/img/bVHBIN)
![clipboard.png](/img/bVHBIX)

                                                图11.3.2

### 小结：
* 三者在创建`bucket`方面类似，而且均可在创建`bucket`时进行相应设置，特别是`访问权限`设置。
* 在初始化权限设置方面，`cos`和`S3`字段基本一致，但是`S3`的`acl`字段可以设置更加丰富的权限值，如`aws-exec-read`,`bucket-owner-read`等。`oss`则只能设置`匿名`访问权限，没有针对特定用户的访问权限设置。

## 十二、获取bucket内的Object列表

### cos 提供获取 bucket 内 Object 列表的 api
* 获取`bucket`内`Object`列表请求语法（如图12.1.1），请求参数（如图12.1.2），响应内容（如图12.1.3）。

![clipboard.png](/img/bVHBQb)

                                                图12.1.1

![clipboard.png](/img/bVHBQm)

                                                图12.1.2

![clipboard.png](/img/bVHBQu)

                                                图12.1.3

### oss 提供获取 bucket 内 Object 列表的 api
* 获取`bucket`内`Object`列表请求语法和`cos`一致，请求参数和`cos`一致，响应内容区别（如图12.2.1）


![clipboard.png](/img/bVHBVG)

                                                图12.2.1


### S3 提供获取 bucket 内 Object 列表的 api
* 获取`bucket`内`Object`列表请求语法和`cos`一致，请求参数和`cos`一致，响应内容区别（如图12.3.1）


![clipboard.png](/img/bVHBWd)

                                                图12.3.1

### 小结：
* 对比`cos`和`oss`以及`S3`的`api`可以看出，在获取`bucket`的`Object`方面，三者均可以指定一些筛选条件，如定界符（`delimiter`）、搜索前缀（`prefix`）、编码方式（`encoding-type`）、条目起点（`marker`，相当于`S3`的`continuation-token`）等。`S3`还可以提供一些额外的请求参数，如用户信息（`fetch-owner`）等，以获取更加详细的返回信息。
* 在返回的信息方面，`cos`、`oss`和`S3`基本一致，唯一的区别是`cos`提供`NextMarker`来标注下一次请求`Object`条目的起点，而`oss`不提供，但是可以根据当前条目起点（`Marker`）和元数据条目数量推算，`S3`则由返回的`NextContinationToken`提供。

## 十三、获取 bucket 具体信息

### cos 不提供获取 bucket 具体属性信息的 api

### oss 提供获取 bucket 具体属性信息的 api
* 请求语法（如图13.2.1），请求响应元素（如图13.2.2）


![clipboard.png](/img/bVHB0g)

                                                图13.2.1

![clipboard.png](/img/bVHB0k)

                                                图13.2.2

### S3 没有提供针对某个 bucket 的信息，但是提供 Get Service 获取用户所有 bucket 信息

### 小结：
* `cos`可以考虑提供一个整合特定`bucket`信息（拥有者、权限设置、地理位置、创建时间、修改时间、内网外网访问域名，CDN加速域名和权限等）。

## 十四、S3 特有的 bucket api
* 由于`S3`的`api`比较完善而且功能丰富，这里特地选择其中比较实用的介绍，包括
    * 事件通知
    * 跨区域复制
    * bucket策略
    * 版本控制

### bucket 事件通知
* 通过`S3`的事件通知设置，用户可以在`bucket`发生某些特定事件时接收消息，不过需要事先设置订阅的事件类型以及事件订阅的接收方。
* 具体操作查看链接：
http://docs.aws.amazon.com/zh_cn/AmazonS3/latest/dev/NotificationHowTo.html
* `oss`也提供事件通知功能，但是没有开放设置和获取事件通知设置的`api`，`S3`提供设置和获取`事件通知`的`api`（`PUT Bucket notification` 和 `GET Bucket notification`），但是没有提供关闭的`api`。

### bucket 跨区域复制
* 该功能允许用户跨不同 `AWS`区域中的存储桶自动、异步地复制对象。可以选择指定两个不同区域（`region`）的`bucket`，将一个`bucket`中的`修改操作`同步到另一个`bucket`中。
* 该功能的使用场景比如：A用户在洛杉矶，A服务的客户主要在东京，A的`静态资源`部署在东京园区的`bucket`中。A希望修改`bucket`中的文件，但是上传文件到东京园区的`bucket`耗时太长，修改效率低下。此时A可以设置跨区域复制，直接关联洛杉矶园区的某个`bucket`和东京园区的`bucket`，上传文件到洛杉矶园区的`bucket`，然后将操作异步到东京园区的bucket中。A仅需维护`静态资源的副本`即可。
* 具体操作查看链接：
http://docs.aws.amazon.com/zh_cn/AmazonS3/latest/dev/crr.html
* `oss`也提供跨区域复制功能，但是没有开放设置和获取跨区域复制的`api`，`S3`提供设置和获取`跨区域复制`的api（`PUT Bucket replication`和 `GET Bucket replication`），也提供关闭的api（`DELETE Bucket replication`）。

### bucket 策略
* `S3`的访问权限控制分为两种策略
    * 基于资源，如`bucket策略`和`acl策略`
    * 基于用户，如`用户策略`
* `bucket策略`是指通过`策略json文本`，设置`bucket`对于不同访问情况的`权限控制`。具体访问情况在介绍防盗链时候已经提及，具体操作查看链接：
http://docs.aws.amazon.com/zh_cn/AmazonS3/latest/dev/example-bucket-policies.html
* `S3`提供设置、获取和关闭`bucket策略`的`api`，设置`bucket策略`请求（如图14.3.1），获取`bucket策略`请求（如图14.3.2），关闭`bucket策略`请求（如图14.3.3）


![clipboard.png](/img/bVHB9m)

                                                图14.3.1

![clipboard.png](/img/bVHB9z)

                                                图14.3.2

![clipboard.png](/img/bVHB9E)

                                                图14.3.3


### bucket 版本控制
* `S3`的版本控制主要用于单文件的多个版本的管理，`S3`提供的版本控制`api`可以选择开启或者关闭版本管理。设置版本控制请求（如图14.4.1），获取版本控制请求（如图14.4.2）。值得一提的是，用户进行跨区域复制时，需要先设置版本控制。

![clipboard.png](/img/bVHCnF)

                                                图14.4.1

![clipboard.png](/img/bVHCnG)

                                                图14.4.2
 
## 十五、Object 单次删除
### cos 提供删除Object的api
* 删除`Object`请求（如图15.1.1）


![clipboard.png](/img/bVHCoF)

                                                图15.1.1

### oss 提供删除Object 的 api
* 删除`Object`请求（如图15.2.1）

![clipboard.png](/img/bVHCo4)

                                                图15.2.1
                                                
### S3 提供删除Object 的 api
* 删除`Object`请求（如图15.3.1）

![clipboard.png](/img/bVHCpx)

                                                图15.3.1

### 小结：
* `cos`、`oss`和`S3`在删除`Object`的`api`方面基本一致，唯一的区别在于`S3`删除`Object`时，会返回被删除`Object`的版本编号以及一个删除标志（如图15.3.2）


![clipboard.png](/img/bVHCpS)

                                                图15.3.2


## 十六、Object批量删除
### cos 不支持Object的批量删除
### oss 提供Object批量删除的api
* 请求语法（如图16.2.1）
* 请求元素（如图16.2.2）
* 响应元素（如图16.2.3）
* 批量删除最多支持一次性删除1000个Object


![clipboard.png](/img/bVHCqw)

                                                图16.2.1

![clipboard.png](/img/bVHCqA)

                                                图16.2.2

![clipboard.png](/img/bVHCqD)

                                                图16.2.3

### S3 提供Object批量删除的api
* 请求语法（如图16.3.1）
* 请求元素（如图16.3.2）
* 响应元素（部分，如图16.3.3）


![clipboard.png](/img/bVHCrc)
![clipboard.png](/img/bVHCrq)

                                                图16.3.1


![clipboard.png](/img/bVHCrx)
![clipboard.png](/img/bVHCrw)

                                                图16.3.2

![clipboard.png](/img/bVHCrD)

                                                图16.3.3

### 小结：
* `oss`在批量删除`Object`方面和`S3`近似。`S3`和`oss`在批量删除当面最大的区别在于，`S3`需要考虑`Object`的版本控制，返回的`xml`数据中含有`Object`版本信息（如果开启了版本控制，如图16.3.4），字段名为`VersionId`。

![clipboard.png](/img/bVHCsk)

                                                图16.3.4

## 十七、获取Object
### cos 提供获取Object的api
* 该操作需要对Object的读权限
* 请求语法（如图17.1.1）
* 请求头部（如图17.1.2），如果bucket为公有读权限，则无需写入签名。
* 响应头部（如图17.1.3）


![clipboard.png](/img/bVHCwn)

                                                图17.1.1

![clipboard.png](/img/bVHCwo)
![clipboard.png](/img/bVHCwq)


                                                图17.1.2

![clipboard.png](/img/bVHCwp)

                                                图17.1.3

### oss 提供获取Object的api
* 该操作需要对Object的读权限
* 请求语法（如图17.2.1）
* 请求头部（如图17.2.2），在设置`oss`头部的时候，需要写入`签名`，否则无效。
* 响应头部（如图17.2.3）


![clipboard.png](/img/bVHDcp)

                                                图17.2.1

![clipboard.png](/img/bVHDcr)

                                                图17.2.2

![clipboard.png](/img/bVHDcs)

                                                图17.2.3

### S3 提供获取Object的api
* 该操作需要对Object的读权限
* 请求语法和`oss`一致
* 请求头部和`oss`一致，在设置`S3`头部的时候，也需要写入`签名`。
* 响应头部和`oss`一致

###小结：
* 在获取`Object`方面，`cos`、`oss`和`S3`近似，但是在设置了`公有读`的情况下，`cos`没有规定设置请求头部需要携带`签名`，但是`oss`和`S3`均需要，否则请求头部设置将会失效。

## 十八、 Object的访问权限

### cos 提供Object访问权限设置和获取的api
* 设置`Object`访问权限请求语法（如图18.1.1），请求头部（如图18.1.2），请求内容（如图18.1.3）。
* 获取`Object`访问权限请求语法（如图18.1.4），响应内容（如图18.1.5）。


![clipboard.png](/img/bVHDdU)

                                                图18.1.1

![clipboard.png](/img/bVHDdV)

                                                图18.1.2

![clipboard.png](/img/bVHDdX)

                                                图18.1.3

![clipboard.png](/img/bVHDd0)

                                                图18.1.4

![clipboard.png](/img/bVHDd1)

                                                图18.1.5

### oss 提供Object访问权限设置和获取的api
* 设置`Object`访问权限请求语法（如图18.2.1）
* 获取`Object`访问权限请求语法（如图18.2.2），响应元素（如图18.2.3）。


![clipboard.png](/img/bVHDeV)

                                                图18.2.1

![clipboard.png](/img/bVHDeX)

                                                图18.2.2

![clipboard.png](/img/bVHDe2)

                                                图18.2.3

### S3 提供 Object 访问权限设置和获取的api
* 设置`Object`访问权限请求语法（如图18.3.1），请求头部（如图18.3.2），请求内容（如图18.3.3）。
* 获取`Object`访问权限请求语法（如图18.3.4），响应内容和请求内容中元素一致。


![clipboard.png](/img/bVHDgk)

                                                图18.3.1

![clipboard.png](/img/bVHDgl)

                                                图18.3.2

![clipboard.png](/img/bVHDgm)

                                                图18.3.3

![clipboard.png](/img/bVHDgp)

                                                图18.3.4

### 小结：
* 从`Object`访问权限`api`而言，`S3`和`cos`做的更好，因为`oss`在设置`Object`的访问权限方面不够细致，只能进行简单的权限设置（读写的共有或私有），不能针对`用户级别`进行权限设置（如授予特定用户读写权限等），和`bucket`权限设置类似。
* 从控制台而言，`cos`和`oss`均只提供用户进行简单访问权限设置的功能（是否`继承bucket权限`，读写的`公有私有`），cos的权限设置请求抓包（如图18.1.6），oss的权限设置请求抓包（如图18.2.4）。而S3提供更丰富的权限设置，具体请求抓包（如图18.3.5），错误授权时报错响应（这里只显示被授予的aws Id不存在错误，如图18.3.6）


![clipboard.png](/img/bVHDg9)

                                                图18.1.6

![clipboard.png](/img/bVHDhc)

                                                图18.2.4

![clipboard.png](/img/bVHDhd)

                                                图18.3.5

![clipboard.png](/img/bVHDhe)
![clipboard.png](/img/bVHDhg)

                                                图18.3.6

## 十九、Object 上传（非分片）


### cos提供上传文件api，而且有两种api可供选择
* `PUT`请求上传文件请求语法（如图19.1.1），请求头部（如图19.1.2），权限相关头部（如图19.1.3），响应头部（如图19.1.4）。
* `POST`请求上传文件请求语法（如图19.1.5），请求参数（如图19.1.6），响应元素（如图19.1.7）。
* 上传`Object`需要获取对应`bucket`的写权限。


![clipboard.png](/img/bVHDho)

                                                图19.1.1

![clipboard.png](/img/bVHDhq)
![clipboard.png](/img/bVHDhs)

                                                图19.1.2

![clipboard.png](/img/bVHDhu)

                                                图19.1.3

![clipboard.png](/img/bVHDhv)

                                                图19.1.4

![clipboard.png](/img/bVHDhz)

                                                图19.1.5

![clipboard.png](/img/bVHDhA)

                                                图19.1.6

![clipboard.png](/img/bVHDhB)

                                                图19.1.7

### oss 提供上传文件api，有两种api可供选择
*  `PUT`请求上传文件请求语法（如图19.2.1），请求头部（如图19.2.2）。
* `POST`请求
    * 请求语法（如图19.2.3），表单域（如图19.2.4）。
    * `oss`提供额外的`POST api`的作用主要是方便浏览器上传文件，在消息正文中以`FormData`方式传输。
    * 在进行`POST`请求时，`oss`还提供`POST policy`设置，用于验证表单的合法性，例如上传文件的`大小限制`，上传成功后的`跳转地址`等（如图19.2.5和图19.2.6）。
    
![clipboard.png](/img/bVHDj4)

                                                图19.2.1

![clipboard.png](/img/bVHDj7)

                                                图19.2.2

![clipboard.png](/img/bVHDkd)

                                                图19.2.3

![clipboard.png](/img/bVHDkf)

                                                图19.2.4

![clipboard.png](/img/bVHDkM)

                                                图19.2.5

![clipboard.png](/img/bVHDmQ)

                                                图19.2.6


### S3 提供上传文件api，有两种api可供选择
* PUT 和 POST 两种，这里只讨论`S3`和`oss`的区别
    *  S3提供了3个额外的头部，分别用于设置Object的存储类型（标准、低频等）、标签、静态网站重定向地址（如图19.3.1）        


![clipboard.png](/img/bVHDnE)
![clipboard.png](/img/bVHDnF)


                                                图19.3.1


### 小结：
* 在单文件非分片上传`api`方面，`cos`只提供了简易的头部设。`oss`和`S3`均提供的参数设置更加丰富。单从`api`角度而言，`cos`请求缺少了如下几种设置：
    * 自定义加密方式（`cos`采用默认`sha1`加密）
    * 存储类型（默认为`标准存储`）
    * 自定义上传成功后的`跳转地址`（上传成功后自动后端重定向）
    * 设置`Object`的访问权限（`cos`的`PUT`请求提供设置，但是`POST`请求不提供）
    * 对表单域的`限制策略`（POST请求）
* 在控制台上传文件方面，`cos`请求抓包（如图19.1.8），`oss`请求抓包（如图19.2.7），`S3`抓包（如图19.3.2）。可以看出`cos`和`oss`在单文件普通上传方面，均使用`POST`请求形式，`S3`则依旧使用`PUT`请求。


![clipboard.png](/img/bVHDoN)


                                                图19.1.18

![clipboard.png](/img/bVHDoT)

                                                图19.2.7

![clipboard.png](/img/bVHDoY)

                                                图19.3.2


## 二十、Object追加上传

### cos 提供文件的追加上传 api
* 需要文件本身允许`追加上传`
* 文件`追加上传 api`请求语法（如图20.1.1），请求参数（如图20.1.2），请求头部（如图20.1.3），响应头部（如图20.1.4）


![clipboard.png](/img/bVHDpE)

                                                图20.1.1

![clipboard.png](/img/bVHDpK)

                                                图20.1.2

![clipboard.png](/img/bVHDpN)
![clipboard.png](/img/bVHDpP)
![clipboard.png](/img/bVHDpQ)

                                                图20.1.3

![clipboard.png](/img/bVHDpS)

                                                图20.1.4
  
### oss 提供文件的追加上传 api
* 需要文件本身允许`追加上传`
* 文件`追加上传 api`请求语法（如图20.2.1），请求头部（如图20.2.2），响应头部（如图20.2.3）


![clipboard.png](/img/bVHDqh)

                                                图20.2.1

![clipboard.png](/img/bVHDql)

                                                图20.2.2

![clipboard.png](/img/bVHDqm)

                                                图20.2.3

### S3 不提供追加上传的 api 接口

### 小结
* 在追加上传`api`方面，`cos`和`oss`近似，唯一的区别在于，`cos`在上传分片时，可以提供追加分片的`sha1`值，而且`hash`方式固定位`RFC 3174`中定义的`160-bit`内容`SHA-1`算法。而`oss`可以指定创建`object`时的服务器端加密编码算法。
* 响应报文头部对比，`cos`可以提供分段的`sha1`校验值，而`oss`不提供。`cos`返回的`ETag`和oss返回的`x-oss-hash-crc64ecma`本质上相同，都是用于（摘要）标志文件。
* `cos`、`oss`和`S3`的控制台均`不提供`文件`追加上传`的功能。


## Object 分块上传
### 大文件的分块上传过程主要分为以下几个逻辑部分：
* 初始化分块上传 
* 分块上传
* 分块上传结束
* 分块上传任务抛弃
* 分块上传列表查询

下面按照这5个部分逐一分析`cos`、`oss`和`S3`在`分块上传api`方面的区别。


## 二十一、初始化分块上传

### cos 提供初始化分块上传 api
* `初始化分块上传`请求语法（如图21.1.1），请求头部（如图21.1.2），其中可以设置一些权限相关头部（如图21.1.3）。响应内容（如图21.1.4）。

![clipboard.png](/img/bVHDtC)

                                                图21.1.1

![clipboard.png](/img/bVHDtI)

                                                图21.1.2

![clipboard.png](/img/bVHDtK)

                                                图21.1.3

![clipboard.png](/img/bVHDtM)

                                                图21.1.4

### oos 提供初始化分块上传 api
* `初始化分块上传`请求语法（如图21.2.1）
* 请求头部和`cos`类似，区别是`oss`提供加密编码算法设置（如图21.2.2）
* 响应内容也和`cos`一致，只是多了编码使用的类型（只有请求时设置了encoding-type 才会返回，如图21.2.3）。
* 初始化分块时，`oss`不提供复杂的访问权限头部设置，但是`cos`提供。


![clipboard.png](/img/bVHDum)

                                                图21.2.1

![clipboard.png](/img/bVHDun)

                                                图21.2.2

![clipboard.png](/img/bVHDup)

                                                图21.2.3
 

### S3 提供初始化分块上传 api
* `初始化分块上传`请求语法（如图21.3.1）
* 请求头部和`cos`类似，也提供访问权限设置。主要区别是`S3`提供`存储类型`设置和`网站重定向`（如图21.3.2）
* 响应内容也和`cos`一致。


![clipboard.png](/img/bVHDuy)

                                                图21.3.1

![clipboard.png](/img/bVHDuD)

                                                图21.3.2

### 小结
* 在初始化分块上传api方面，`cos`和`S3`基本一致，而`oss`则缺少访问权限设置。
* `cos`相比`S3`最主要区别是请求缺少存储类型设置（`标准`、`近线`、`低频`等）。
* 在控制台进行请求抓包对比，`cos`请求（如图21.1.5），`oss`请求（如图21.2.4），`S3`请求（如图21.3.3）。可以看出：
    * `cos`控制台`初始化分块上传`时，需要事先计算每一块的校验值（`datasha`），通过`FormData`传给后台，`oss`和`S3`则不需要事先算出校验值。
    * `S3`的响应头部中，`x-s3-console-ks-session-id`相当于`UploadId`，用于标记这个分块上传任务。`x-s3-console-ks-session-sequencer`记录当前上传的分块的序号，一开始为1，依次累加，返回的字段中没有记录分块校验值的字段。


![clipboard.png](/img/bVHDuY)

                                                图21.1.5

![clipboard.png](/img/bVHDva)

                                                图21.2.4

![clipboard.png](/img/bVHDvy)
![clipboard.png](/img/bVHDvz)

                                                图21.3.3


## 二十二、分块上传

### cos 提供分块上传 api
* 需要在`初始化分块上传`之后使用
* 每次上传分块时，需要携带`partNumber`和`uploadID`，其中`uploadID`为初始化时返回的`uploadID`，用于标志上传任务（文件）。`partNumber`为分块的编号，如果上传了相同的分块（`partNumber`和`uploadID`一致），则新传入分块将覆盖旧的分块。而且支持乱序上传，因此可以进行分块并发上传。支持的块的数量为1到10000，块的大小为1 MB 到5 GB。请求语法（如图22.1.1），请求头部（如图22.1.2）。


![clipboard.png](/img/bVHDwh)

                                                图22.1.1

![clipboard.png](/img/bVHDwo)
![clipboard.png](/img/bVHDws)

                                                图22.1.2

### oss 也提供分块上传 api
* 分块上传过程类似于`cos`，也是通过`uploadID`和`partNumber`标志上传任务和当前分块编号。
* 分块数量最多为 10000，除了最后一块之外均需要大于100KB，对于相同`uploadID`和`partNumber`的块进行覆盖。
* 虽然每一分块均需要大于100KB，但是`oss`在`分块上传结束`之前并不会校验块大小。
* 请求语法（如图22.2.1）


![clipboard.png](/img/bVHDwU)

                                                图22.2.1

### S3 也提供分块上传 api
* 区别在于上传时可以在头部中指定使用的`加密类信息`和`hash`信息，如算法类型，客户加密秘钥，客户加密秘钥的MD5摘要等（如图22.3.1）


![clipboard.png](/img/bVHDxa)

                                                图22.3.1

### 小结
* 在分块上传部分，`cos`、`oss`和`S3`基本没差别，只是在验证方面`S3`做的更严谨，可以在头部中指定使用的`加密类信息`和`hash`信息。
* 在控制台对三者进行请求抓包，cos的请求（如图22.1.3），oss请求（如图22.2.2），S3请求（如图22.3.2），可以看出：
    * `cos`上传分块接口，需要提供文件校验值（`sha`），分片大小（`sliceSize`），上传任务标志（`session`，和`uploadId`作用一致）偏移量（`offset`）。由于提供了`offset`，因此没有提供`partNumber`（分块编号）。
    * `S3`分块上传接口和`cos`原理类似，不过用于标志上传任务的字段放在头部（`x-s3-console-ks-session-id`，功能和`uploadId`一致），用户标记分块编号的字段也在头部（`x-s3-console-ks-session-sequencer`，功能和`partNumber`一致）。
    * `oss`的文件上传接口则比较特殊，本质上而言，控制台只发起了一次`http`请求，而不是像`cos`和`S3`一样每上传一个分块就发起一次`http`请求。原因是`oss`控制台并`不提供`分块上传，这次`http`请求在上传完成之前一直处于`request send`状态。

* 在控制台文件上传方面，`oss`和`cos`区别较大。
    * `oss`和`cos`一样使用`FormData`传输文件和其他字段，但是cos事先对文件进行`分块处理`，每次只传输一个文件块。而`oss`不对文件进行分块，直接上传`整个文件`。
    * 进度显示方面，`oss`通过监听`xhr.upload.onprogress`事件，通过返回的`event.loaded`和文件总大小更新进度条。`cos`则通过上传成功的分块偏移量和文件总大小更新进度。
    * 由于`oss`控制台上传的文件没有使用`分块上传`，因此如果中途`刷新页面`，碎片管理中不会产生对应的`文件碎片`，也无法`续传`。

![clipboard.png](/img/bVHDzE)


                                                图22.1.3

![clipboard.png](/img/bVHDzH)
![clipboard.png](/img/bVHDzI)

                                                图22.2.2


![clipboard.png](/img/bVHDzK)
![clipboard.png](/img/bVHDzL)

                                                图22.3.2


## 二十三、分块上传完成

### cos 提供完成分块上传的 api
* 当所有分块上传完毕后，需要使用该`api`完成上传任务。
* 需要传输每一个分块对应的`PartNumber`和`ETag`（分块的`sha1`校验值），校验成功后才进行分块合并。请求语法（如图23.1.1），请求内容（如图23.1.2），响应内容（如图23.1.3）。

![clipboard.png](/img/bVHDAJ)

                                                图23.1.1

![clipboard.png](/img/bVHDAQ)

                                                图23.1.2

![clipboard.png](/img/bVHDAS)

                                                图23.1.3

### oss 也提供完成分块上传的 api
* 具体过程和`cos`一致，都需要提供每一个分块的`partNumber`和`ETag` ，分片校验之后再进行分块合并。请求语法（如图23.2.1），请求内容（如图23.2.2），响应元素（如图23.2.3）。

![clipboard.png](/img/bVHDBO)

                                                图23.2.1

![clipboard.png](/img/bVHDBP)

                                                图23.2.2
 
![clipboard.png](/img/bVHDBV)

                                                图23.2.3

### S3 也提供完成分块上传的 api
* `S3`提供的完成分块上传`api`和`oss`一致，主要区别在于`S3`返回的头部中含有文件的`版本编号`，`加密算法类型`等（如图23.3.1），请求语法（如图23.3.2）。

![clipboard.png](/img/bVHDCr)

                                                图23.3.1

![clipboard.png](/img/bVHDCs)

                                                图23.3.2

### 小结
* 在完成分块上传`api`方面，三者类似，只是`S3`额外返回新文件的`版本编号`。
* 在控制台请求抓包方面，`cos`请求（如图23.1.4），`oss`请求（如图23.2.4），`S3`请求（如图23.3.3）。
    * `cos` 上传成功后会返回加速域名（`access_url`）、外网域名（`source_url`）等信息。值得注意的是`vid`字段，用于标志文件的`版本编号`。
    * `S3`上传成功后，响应的头部中会缺少`x-s3-console-ks-http-continuation`、`x-s3-console-ks-session-id`、`x-s3-console-ks-session-sequencer`等字段，用于标志分块上传的`任务编号`、`分块编号`等信息，前端收到的响应头部中不含有这些信息，说明分块上传已经完成。
    * `oss`在控制台不提供分块上传，传输时只发起一个`http`请求，当传输成功后，`request send`结束，后台返回文件的`hash`值等信息。

    

![clipboard.png](/img/bVHDzr)
![clipboard.png](/img/bVHDzs)

                                                图23.1.4

![clipboard.png](/img/bVHDzt)
![clipboard.png](/img/bVHDzv)

                                                图23.2.4

![clipboard.png](/img/bVHDD7)
![clipboard.png](/img/bVHDEf)

                                                图23.3.3



## 二十四、分块上传任务抛弃

### cos 提供分块上传任务抛弃的api
* 用于停止分块上传，并且删除已经上传成功的分块。请求语法（如图24.1.1）

![clipboard.png](/img/bVHDQu)

                                                图24.1.1

### oss 也提供分块上传任务抛弃的api
* 请求语法（如图24.2.1）

![clipboard.png](/img/bVHDQD)

                                                图24.2.1
                                                
### S3 也提供分块上传任务抛弃的api
* 请求语法（如图24.3.1）


![clipboard.png](/img/bVHDQI)

                                                图24.3.1

### 小结
* 在分块上传任务抛弃方面，cos、oss、S3一致。


## 二十五、分块上传列表查询

### cos 提供分块上传列表查询 api
* 用于查询特定分块上传任务中，已经成功上传的分块列表。
* 该api主要用于断点续传，先获取已经上传成功的分块信息，然后只上传未成功的分块即可。
* 请求语法（如图25.1.1），请求内容（如图25.1.2），响应内容（如图25.1.3）。


![clipboard.png](/img/bVHDRX)

                                                图25.1.1

![clipboard.png](/img/bVHDSg)

                                                图25.1.2

![clipboard.png](/img/bVHDSk)

                                                图25.1.3

### oss 也提供分块上传列表查询 api
* 具体和`cos`一致（请求内容和返回内容都一致）。请求语法（如图25.2.1）


![clipboard.png](/img/bVHDSE)

                                                图25.2.1

### S3 也提供分块上传列表查询api
* 大致和`cos`一致，不过请求返回内容有不同。S3额外提供几个特殊的字段（如图25.3.1），如：
    * `x-amz-abort-date` ：根据bucket的生命周期规则，该分块被删除的时间。
    * `x-amz-abort-rule-id` ：导致该分块会被删除的生命周期规则。
    * `StorageClass` ：存储类型。
    

![clipboard.png](/img/bVHDSX)
![clipboard.png](/img/bVHDSY)

                                                图25.3.1
                                                
### 小结
* 在分块上传列表查询api方面，三者基本一致，不过`S3`将分块和生命周期结合在一起，处理更细。
* 分块上传列表查询主要用于断点续传，控制台对比方面：
    * oss控制台上传文件没有使用分块上传，不支持断点续传。
    * S3控制台支持分块上传，但是不支持断点续传，控制台上传文件手动中止后，需要重新上传所有分块。
    * cos控制台支持分块上传，也支持断点续传，通过请求抓包（如图25.1.4）可以看出，后台返回了所有上传成功的分块信息。
    
![clipboard.png](/img/bVHDS5)
![clipboard.png](/img/bVHDS6)

                                                图25.1.4

## 二十六、当前分块上传任务列表

### cos 不提供该 api。
### oss 提供当前分块上传任务列表 api
* 该api主要用于查询那些已经进行分块初始化，但是没有上传成功，也没有被中止的任务。
* 该接口可以用来查看当前未结束的任务，判断是否继续执行，是否断点续传等。                        
* 请求语法（如图26.2.1），请求参数（如图26.2.2），响应内容（如图26.2.3）


![clipboard.png](/img/bVHDTr)

                                                图26.2.1

![clipboard.png](/img/bVHDTu)

                                                图26.2.2

![clipboard.png](/img/bVHDTv)

                                                图26.2.3

### S3 不提供该 api。
  
### 小结
* `oss`提供的当前分块上传任务列表api，可以方便用户管理未结束的任务，及时抛弃任务（删除已经上传成功的`文件碎片`），节省存储空间。

## 二十七、文件拷贝

### cos 提供文件拷贝的 api
* `cos`的文件拷贝`api`可以通过`文件移动api`实现，文件移动`api`可以用于文件的重命名和移动，`请求地址`中的参数为源文件的位置，`请求内容`中的参数`dest_fileid`指明移动至的目标路径（以及`新的文件名`）。
* 请求语法（如图27.1.1），请求内容（如图27.1.2），返回内容（如图27.1.3）。


![clipboard.png](/img/bVHEYu)

                                                图27.1.1

![clipboard.png](/img/bVHEYv)

                                                图27.1.2

![clipboard.png](/img/bVHEYx)

                                                图27.1.3

### oss 提供文件拷贝的 api
* 对于小于1GB的文件，可以直接调用`CopyObject` api 
    * 请求时需传入`DestObjectName` （目标文件名）、`DestBucketName`（目标bucket）和`x-oss-copy-source`头部（拷贝源）
    * 请求语法（如图27.2.1），请求头部（如图27.2.2），    响应元素（如图27.2.3） 
* 对于大于1GB的文件，可以调用`UploadPartCopy` api 
    * 在调用该 api 前，需要调用`Initiate Multipart Upload`（初始化分块上传）api，获取`Upload ID`。该api可以视为`分块上传`的特例，相当于将指定bucket中的大文件`分块上传`到另一个bucket中。
    * 请求语法（如图27.2.4），请求头部（如图27.2.5），    响应示例（如图27.2.6） 
    

![clipboard.png](/img/bVHEYY)

                                                图27.2.1

![clipboard.png](/img/bVHEZa)
![clipboard.png](/img/bVHEZk)

                                                图27.2.2

![clipboard.png](/img/bVHEZo)

                                                图27.2.3

![clipboard.png](/img/bVHE0n)

                                                图27.2.4

![clipboard.png](/img/bVHE0t)
![clipboard.png](/img/bVHE0u)

                                                图27.2.5
                                                
![clipboard.png](/img/bVHE0w)

                                                图27.2.6


### S3 提供文件拷贝的 api
* 和`oss`类似，对于`小于5GB`的文件，可以需要调用`CopyObject`api，请求语法和参数和`oss`类似，区别是请求头部可以选择指定`存储类型`和`静态网站重定向`位置（如图27.3.1），响应头部会携带源文件和目标文件的`版本信息`（如图27.3.2）                         
* 对于`大于5GB`的文件，可以直接调用`Upload Part - Copy` api 
    * 和`oss`一样，在调用该 api 前，需要调用`Initiate Multipart Upload`（初始化分块上传）api，获取`Upload ID`才能继续上传。
    * 请求语法和参数与`oss`类似，区别在于会响应头部含有`版本信息`和`服务器端加密算法`等信息（如图27.3.3）


![clipboard.png](/img/bVHE2H)
![clipboard.png](/img/bVHE2K)

                                                图27.3.1

![clipboard.png](/img/bVHE2O)
![clipboard.png](/img/bVHE2R)

                                                图27.3.2

![clipboard.png](/img/bVHE3c)

                                                图27.3.3

### 小结
*  `oss`和`S3`提供的文件拷贝api对大文件和小文件分开处理，更加细致。
*  `oss`和`S3`提供更加多样化的文件指定，如`copy-source-if-match`、`copy-source-if-unmodified-since`、`metadata-directive`等，可以更精确的限定拷贝场景。
* `S3`和`oss`的最大区别在于可以指定`存储类型`、`网站重定向`等信息，可以返回源文件和目标文件的`版本信息`等。
                                

## 总结
* 在`api`方面而言，`cos`提供`api`相对较少，有些功能暂不提供，如`跨区域复制`、`批量操作`（批量删除和设置）、`访问日志`和各类`bucket属性设置`（`防盗链`、`CORS跨域设置`、`静态网站托管`等，不提供`api`，可以在`控制台`进行操作）。`cos`大多数功能都能在控制台进行相应设置， 而且操作相当方便。特别是在`文件上传`方面，`cos`控制台是`web端`表现最好的一个，`分块上传`和`断点续传`的支持相当到位。
*  `oss`和`S3`提供的`api`有很多相似的地方，功能上十分丰富，其中`S3`更加细致，提供更加全面的`访问权限`设置、`版本管理`以及`存储类型`设置，但是`S3`在易用性方面欠佳，很多需要配置相应的`xml`文本或者`JSON document`，特别是在配置`bucket策略`文本的时候略微繁琐。


  [1]: /img/bVHuBn
  [2]: /img/bVHuCN
  [3]: /img/bVHuDn
  [4]: /img/bVHuDy
  [5]: /img/bVHuD3
  [6]: /img/bVHuD9
  [7]: /img/bVHuEn
  [8]: /img/bVHuEt
  [9]: /img/bVHvhw
  [10]: /img/bVHvhJ
  [11]: /img/bVHvyd
  [12]: /img/bVHvzM
  [13]: /img/bVHvzQ
  [14]: /img/bVHvDS
  [15]: /img/bVHvJn
  [16]: /img/bVHvJH
  [17]: /img/bVHvRp
  [18]: /img/bVHvRK
  [19]: /img/bVHvRN
  [20]: /img/bVHvSF
  [21]: /img/bVHvVC
  [22]: /img/bVHvXi
  [23]: /img/bVHvXj
  [24]: /img/bVHvYw
  [25]: /img/bVHvYx
  [26]: /img/bVHAkK
  [27]: /img/bVHAkQ
  [28]: /img/bVHAkT
