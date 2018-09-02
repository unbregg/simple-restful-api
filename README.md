# RESTful API 最佳实践

- [基本](#基本)
	- [HTTP请求方法规则](#http请求方法规则)
	- [返回数据格式](#返回数据格式)
	- [HTTP状态码约定](#http状态码约定)
- [获取数据](#获取数据)
	- [请求资源](#请求资源)
	- [过滤](#过滤)
	- [排序](#排序)
	- [分页](#分页)
- [资源的增删改](#资源的增删改)
	- [新增资源](#新增资源)
	- [更新资源](#更新资源)
	- [删除资源](#删除资源)
- [Error对象](#error对象)
- [登录授权约定](#登录授权约定)

## 基本

### HTTP请求方法规则

对于资源的具体操作类型，由HTTP动词表示。
常用的HTTP动词有下面四个（括号里是对应的SQL命令）。

>GET（SELECT）：从服务器取出资源（一项或多项）。

>POST（CREATE）：在服务器新建一个资源。

>PUT（UPDATE）：在服务器更新资源。

>DELETE（DELETE）：从服务器删除资源。

>PATCH（UPDATE）：部分更新资源

下面是一些例子。

>GET /users：列出所有用户

>POST /users：新建一个用户

>GET /users/ID：获取某个指定用户的信息

>PUT /users/ID：更新某个指定用户的信息

>DELETE /users/ID：删除某个用户

>GET /users/ID/roles ：列出某个指定用户下的所有角色

>GET /users/ID/roles/ID：获取某个指定用户下的某个角色

### 返回数据格式

为了保障前后端的数据交互的顺畅，建议规范数据的返回，并采用固定的数据格式封装（content-type 为 json）。

```
{
    // 返回具体的业务数据，单个资源返回Object类型，资源集合返回Array类型
    // data和error不能同时存在。
    data: {},
    // 错误对象
    error: {
	    // 错误码
	    code: '', 
	    // 错误描述，面向开发者
	    description: '错误描述',
	    // 错误信息，面向终端用户
	    message: '出错信息'
    },
    // 当请求资源集合，需要分页时，会存在分页对象
    pagination: {
	    number: 1,
	    size: 10,
	    total: 100
	},
    // 存储额外的信息
    meta: {}
}
```

### HTTP状态码约定

Code | Meaning
--------------------------|----------------------------
200 OK | 请求成功。
201 Created | 请求成功并且服务器创建了新的资源。
400 Bad Request | 服务器不理解请求的语法(例如传参错误)。
401 Unauthorized | 授权信息非法或未送达。
403 Forbidden | 已授权登陆的用户无权限访问该接口。
404 Not Found | 资源未找到。
500&nbsp;Internal&nbsp;Server&nbsp;Error | 服务器内部错误。

## 获取数据

### 请求资源

在请求资源时，需使用`GET`方式向服务端发起请求。
例如需要获取`user`集合时：
```
GET /users HTTP/1.1
content-type: application/json

response
{
	data: [
		{id: 1, name: '小红', sex: '女'},
		{id: 2, name: '小明', sex: '男'}
	]
}
```
获取`id`为1的`user`:
```
GET /users/1 HTTP/1.1
content-type: application/json

response
{
	data: {id: 1, name: '小红', sex: '女'}
}
```

### 过滤

当要求服务端根据某些条件过滤出资源集合时，需将过滤条件转成字符串放在`filter`中，各条件之间是`与`操作，例如需要过滤出`姓名为小红，性别为女`的集合：
```
GET /users?filter={"name":"小红","sex":"女"} HTTP/1.1
content-type: application/json

response
{
	data: [{id: 1, name: '小红', sex: '女'}]
}
```

### 排序

对`user`资源集合进行按`name`降序排序
```
GET /users?orders=name.desc  HTTP/1.1
```
对`user`资源集合先按name降序排序，再按age升序
```
GET /users?orders=name.desc,age.asc  HTTP/1.1
```

### 分页

需要服务端进行数据的分页处理时，需将分页信息转成字符串放在`page`中。
例如需要请求第一页，数量为10的`user`：
```
GET /users?page={"number":1,"size":10} HTTP/1.1
content-type: application/json

response
{
	data: [
		{id: 1, name: '小红', sex: '女'},
		...
		{id: 10, name: '小绿', sex: '男'}
	],
	pagination: {
		number: 1,
		size: 10, 
		total: 100
	}
}
```
此时发现在返回的数据中多了一个`pagination`对象，它记录着当前页码`number`，单页数量`size`，以及总数`total`。

## 资源的增删改

### 新增资源

**Request**

新增资源时，需要发起`POST`类型的请求，并将请求数据放在`data`字段中。
例如新增一个`user`：
```
POST /users HTTP/1.1
content-type: application/json

request
{
	data: {name: '小红', sex: '女'}
}
```

**Response (success)**

*201 Created*
新建成功，并且服务端需要返回完整的资源对象<sup>[1]</sup>。
```
response
{
	data: {id:1, name: '小红', sex: '女'}
}
```


### 更新资源

更新资源可以通过两种方式，`PUT`和`PATCH`。两者的区别在于，`PUT`需要将完整的资源对象发送给后端，否则服务端将会对遗漏掉的属性做置null处理。而`PATCH`只需发送将要修改的属性。
例如，将`id`为1的`user`的`sex`改为`男`：
```
PUT /users/1 HTTP/1.1
content-type: application/json

request
{
	data: {id: 1, name: '小红', sex: '男'}
}
```

```
PATCH /users/1 HTTP/1.1
content-type: application/json

request
{
	data: {sex: '男'}
}
```

**Response (success)**

*200 OK*
资源更新成功，同时服务端需返回完整的资源对象<sup>[1]</sup>。


### 删除资源

删除资源需通过`DELETE`发起，例如删除`id`为1的`user`：
```
DELETE /users/1 HTTP/1.1
```
**Response (success)**

*200 OK*
资源删除成功，服务端仅需返回一个空对象。
```
{}
```

## Error对象

当服务端响应的状态码为`40x`或者`50x`时，建议返回一个Error对象，方便开发者排查问题，及友好地向用户提示。
其中，服务端对提交的数据校验不通过时（例如唯一性校验），需返回`422`状态码：
```
{
	code: 'validationError',
	description: '',
	message: '名称重复了'
}
```

## 登录授权约定
统一使用基于OA的[SSO单点登录方案](http://cf.meitu.com/confluence/pages/viewpage.action?pageId=50871007)

---------------------------------------
**标注**：
1.创建或更新资源成功需要返回完整资源对象的原因在于，有些属性值服务端会按照一定的业务逻辑去更新，而不需要客户端发送该属性的值，例如`updateTime`。

**参考资料**
1. [JSON API](http://jsonapi.org/format/)
2. [RESTful API Design](https://blog.philipphauer.de/restful-api-design-best-practices/)
3. [Microsoft api-guidelines](https://github.com/Microsoft/api-guidelines/blob/vNext/Guidelines.md)
4. [REST HTTP status codes for failed validation](https://stackoverflow.com/questions/3290182/rest-http-status-codes-for-failed-validation-or-invalid-duplicate)
5. [API接口规范](https://github.com/mishe/blog/issues/129)
