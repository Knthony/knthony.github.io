---
title:  利用Rust构建一个REST API服务
date: 2020/3/10 19:04:23
categories:
- Tech
tags:
- rust
- api
---



Rust 是一个拥有很多忠实粉丝的编程语言，还是很难找到一些用它构建的项目，而且掌握起来甚至有点难度。

想要开始学习一门编程语言最好的方式就是利用它去做一些有趣的项目，或者每天都使用它，如果你的公司在构建或者已经在使用微服务了，可以尝试一下使用rust把它重写一遍。



第一次使用Rust的时候，和学习其他语言一样，需要了解基础知识，在你了解基础语法和一些常用的概念后，就可以思考如何使用Rust进行异步编程了，很多编程语言都在运行时内置了异步的运行模式，比如在一些发送请求或者等待后台运行的场景里面会使用到异步编程。



补充一点，Tokio是一个事件驱动的非阻塞I / O平台，用于使用Rust编程语言编写异步应用程序,很有可能你下一家公司就在用它，由于你会选择一些内置了Tokio的库来编写异步程序，因此接下来，我们会使用wrap来实现一个异步API平台。



## 创建项目

跟着下面的教程，你需要装下面列出的库：

- Wrap  用来创建API服务
- Tokio 运行异步服务器
- Serde 序列化输入的JSON
- parking_lot 为存储器提供读写锁

首先，使用Cargo创一个新项目

`cargo new neat-api --bin`

将上面需要安装的依赖库添加到Cargo.toml文件里

```rust
…
[dependencies]
warp = "0.2"
parking_lot = "0.10.0"
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "0.2", features = ["macros"] }
```

为了第一次运行测试，在`main.rs`创建一个“Hello World” 例子

```rust
use wrap:: Filter;

#[tokio::main]
async fn main() {
  // GET /hello/warp => 200 OK with body "Hello, warp!"
  let hello = wrap::path!("hello" / String)
  		.map(|name| format!("Hello, {}", name));
  
  warp::serve(hello)
  		.run(([127,0,0,1], 3030))
  		.await;
}
```

`Filters`  是解析请求和匹配路由的方法，使用`cargo run` 来启动服务，在浏览器中输入`http://localhost:3030/hello/WHATEVER`, warp将通过Filiters发送请求，然后执行触发请求。



在`let hello=...`,我们创建了一个新的路由, 意味着所有通过/hello的请求都由该方法处理，因此，它将返回Hello， WHATEVER。

如果我们在浏览器访问`http://localhost:3030/hello/new/WHATEVER`,程序将返回404，因我们没有定义/hello/new + String 相关的过滤器。



## 创建API



接下来，我们创建一个真正的API来演示上面介绍的概念，用API实现一个购物车列表是很好的例子，我们可以在列表里添加条目，更新或者删除条目，还可以看到整个列表，因此，我们利用`GET`,`DELETE`,`PUT`,`POST`HTTP方法实现四个不同的路由。



## 创建本地存储



除了路由，还需要将状态存储在文件或局部变量中，在异步环境中，我们需要确保在访问存储器的同一时间只有一个状态，因此，线程之间不能存在不一致的情况，在Rust中，利用`Arc`可以让编译器知道何时该删除值，何时控制读写锁（`RwLock`），这样，不会有两个参数在不同的线程上操作同一块内存。

如下是实现方法：

```rust
use parking_lot::RwLock;
use std::collections::HashMap;
use std::sync::Arc;

type Items = HashMap<String, i32>;

#[derive(Debug, Deserialize, Serialize, Clone)]
struct Item {
    name: String,
    quantity: i32,
}

#[derive(Clone)]
struct Store {
  grocery_list: Arc<RwLock<Items>>
}

impl Store {
    fn new() -> Self {
        Store {
            grocery_list: Arc::new(RwLock::new(HashMap::new())),
        }
    }
}
```

`Posting`一个条目到列表里

现在我们添加第一个路由，添加条目到列表里，将POST参数请求添加进路由地址中，我们的应用需要返回一个HTTPcode让请求者知道他们是否请求成功，wrap通过HTTP库提供了一个基础形式，这些我们也要添加进去。

`use warp::{http, Filter};`

POST方法的请求方法如下所示

```rust
async fn add_grocery_list_item(
		item: Item,
  	store: Store
  	) -> Result<impl warp::Reply, warp::Rejection> {
      store.grocery_list.write().insert(item.name, item.quantity);
      
      Ok(warp::reply::with_status(
        "Added item to the grocery list",
        http::StatusCode::CREATED,
      ))
}

```

warp框架提供了一个`reply with status`的选项，因此我们可以添加文本以及标准的HTTP状态码，以便让请求者知道是否请求成功或者需要再次请求一次。



现在增加新的路由地址，并调用上面写好的方法。由于你可以为这个方法调用JSON，需要新建一个`json_body`功能函数将`Item`从请求的`body`中提取出来。

额外我们需要将存储方法通过克隆传递给每一个参数，创建一个warp filter。

```rust
fn json_body() -> impl Filter<Extract = (Item,), Error = warp::Rejection> + Clone{
	    // When accepting a body, we want a JSON body
    	// (and to reject huge payloads)...
		warp::body::content_length_limit(1024*16).and(warp::body::json())
}

#[tokio::main]
async fn main() {
  let store = Store::new();
  let store_filter = warp::any().map(move || store.clone());
  let add_items = warp::post()
  		.and(warp::path("v1"))
  		.and(warp::path("groceries"))
  		.and(warp::path.end())
  		.and(json_body())
  		.and(store_filter.clone())
  		.and_then(add_grocery_list_item);
  
  warp::serve(add_items)
  		.run(([127,0,0,1], 3030))
  		.await;
}
```

接下来通过curl或者Postman通过POST请求测试服务，现在它将是一个可以独立处理HTTP请求的服务了，通过

`cargo run`启动服务，然后在新的窗口中运行下面的命令。

```
curl --location --request POST 'localhost:3030/v1/groceries' \
--header 'Content-Type: application/json' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "name": "apple",
    "quantity": 3
}'
```



## 获取列表

现在我们可以通过post将条目添加到列表中，但是我们不能检索它们，我们需要为`GET`新建另一个路由，我们不需要为这个路由解析Json。



```rust
#[tokio::main]
async fn main() {
		let store = Store::new();
  	let store_filter = warp::any().map(move || store.clone());
  	
  	let add_items = warp::post()
  			.and(warp::path("v1"))
  			.and(warp::path("groceries"))
  			.and(warp::path::end())
  			.and(json_body())
  			.and(store_filter.clone())
  			.and_then(add_grocery_list_item);
  	
  	let get_items = warp::get()
  			.and(warp::path("v1"))
  			.and(warp::path("groceries"))
  			.and(warp::path::end())
  			.and(store_filter.clone())
  			.and_then(get_grocery_list);
  	
  	let routes = add_items.or(get_items);
  	
  	warp::serve(routes)
  			.run(([127,0,0,1],3030))
  			.await;
}

```



当你研究`Arc`背后的数据结构时，你会体会到到异步的存在，你将需要先对`RwLock`中的数据先进行`.read()`然后进行`.iter()`操作，因此新建一个变量用来返回给请求者，由于Rust所有权模型的性质，您不能简单地阅读并返回底层的杂货清单。方法如下：

```rust
async fn get_grocery_list(
    store: Store
    ) -> Result<impl warp::Reply, warp::Rejection> {
        let mut result = HashMap::new();
        let r = store.grocery_list.read();


        for (key,value) in r.iter() {
            result.insert(key, value);
        }

        Ok(warp::reply::json(
            &result
        ))
}
```



最后缺少的两个方法是`UPDATE`和`DELETE`。 对于`DELETE`，可以直接复制`add_grocery_list_item`，但是要使用`.remove（）`代替`.insert（）`。

一种特殊情况是`update`。 Rust HashMap实现也使用.insert（），但是如果键不存在，它会更新值而不是创建新条目。

因此，只需重命名该方法，然后为POST和PUT调用它即可。

对于DELETE方法，只需要传递项目的名称，创建一个新结构，并为该新类型添加另一个parse_json（）方法。

可以简单地重命名add_grocery_list_item方法以将其命名为update_grocery_list，并为`warp :: post（）`和`warp :: put（）`对其进行调用。 您的完整代码应如下所示：

```rust
use warp::{http, Filter};
use parking_lot::RwLock;
use std::collections::HashMap;
use std::sync::Arc;
use serde::{Serialize, Deserialize};

type Items = HashMap<String, i32>;

#[derive(Debug, Deserialize, Serialize, Clone)]
struct Id {
    name: String,
}

#[derive(Debug, Deserialize, Serialize, Clone)]
struct Item {
    name: String,
    quantity: i32,
}

#[derive(Clone)]
struct Store {
  grocery_list: Arc<RwLock<Items>>
}

impl Store {
    fn new() -> Self {
        Store {
            grocery_list: Arc::new(RwLock::new(HashMap::new())),
        }
    }
}

async fn update_grocery_list(
    item: Item,
    store: Store
    ) -> Result<impl warp::Reply, warp::Rejection> {
        store.grocery_list.write().insert(item.name, item.quantity);


        Ok(warp::reply::with_status(
            "Added items to the grocery list",
            http::StatusCode::CREATED,
        ))
}

async fn delete_grocery_list_item(
    id: Id,
    store: Store
    ) -> Result<impl warp::Reply, warp::Rejection> {
        store.grocery_list.write().remove(&id.name);


        Ok(warp::reply::with_status(
            "Removed item from grocery list",
            http::StatusCode::OK,
        ))
}

async fn get_grocery_list(
    store: Store
    ) -> Result<impl warp::Reply, warp::Rejection> {
        let mut result = HashMap::new();
        let r = store.grocery_list.read();


        for (key,value) in r.iter() {
            result.insert(key, value);
        }

        Ok(warp::reply::json(
            &result
        ))
}

fn delete_json() -> impl Filter<Extract = (Id,), Error = warp::Rejection> + Clone {
    // When accepting a body, we want a JSON body
    // (and to reject huge payloads)...
    warp::body::content_length_limit(1024 * 16).and(warp::body::json())
}

fn post_json() -> impl Filter<Extract = (Item,), Error = warp::Rejection> + Clone {
    // When accepting a body, we want a JSON body
    // (and to reject huge payloads)...
    warp::body::content_length_limit(1024 * 16).and(warp::body::json())
}


```



测试脚本如下：



POST

```
curl --location --request POST 'localhost:3030/v1/groceries' \
--header 'Content-Type: application/json' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "name": "apple",
    "quantity": 3
}'
```



UPDATE

```
curl --location --request PUT 'localhost:3030/v1/groceries' \
--header 'Content-Type: application/json' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "name": "apple",
    "quantity": 5
}'
```



GET

```
curl --location --request GET 'localhost:3030/v1/groceries' \
--header 'Content-Type: application/json' \
--header 'Content-Type: text/plain'
```



DELETE

```
curl --location --request DELETE 'localhost:3030/v1/groceries' \
--header 'Content-Type: application/json' \
--header 'Content-Type: text/plain' \
--data-raw '{
    "name": "apple"
}'
```



>翻译自 Creating a REST API in Rust with warp，作者：Bastian Gruber
>
>原文链接： https://blog.logrocket.com/creating-a-rest-api-in-rust-with-warp/