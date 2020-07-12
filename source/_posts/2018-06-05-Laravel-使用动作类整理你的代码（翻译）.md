---
title: 'Laravel : 使用动作类整理你的代码（翻译）'
date: 2018-06-05 17:25:23
categories:
- '翻译'
tags:
- 'PHP'
- '技术'
---
友偶阅网文一篇，头中尾英，阅不能，遂中译之。原文链接 ：[Keeping your Laravel applications DRY with single action classes](https://medium.com/@remi_collin/keeping-your-laravel-applications-dry-with-single-action-classes-6a950ec54d1d)

“这段代码该放在哪里？”恐怕是在谈论应用结构时最经常提起的问题。“我应该放在 Controller 里吗？还是 Model ？还是哪里？”，虽然 Laravel 是个十分灵活的框架，但要解答这个问题也不总能简单明了。

当你知道你的应用程序只有一个接入点的时候，把逻辑写在 Controller 是完全没问题的。但如今应用一般都会有好几个接入点共用同一个功能。

例如，很多应用都会有一个用户注册的表单，提交到 Controller ，然后根据是否成功注册来返回有用的信息。在移动端应用的场景下，一般都会有个使用 JSON 格式返回的 API 专门用于用户注册。而且利用 Laravel 的 artisan 命令来创建用户也很常见，尤其是在项目前期的开发阶段。

```php
class UserController {
  public function create(CreateUserRequest $request){
    $user = User::create($request->all());
    
    return view('user.create',['user' = > $user]);
  }
}

class UserApiController {
  public function create(CreateUserRequest $request){
    $user = User::create($request->all());
    
    return response()->json($user);
  }
}
```



些重复代码的确看上去没什么毛病，但如果就这样让业务逻辑继续扩充下去，例如：你希望给新注册用户发一封提醒邮件，那么你需要在两个 Controller 里同时加上这个逻辑。所以为了代码能够整洁优雅起来，我们需要把它放到其他地方去。

## 别把服务类神化了

一开始，可以先临时创建一个类，把所有针对某个业务逻辑的代码整理起来，例如：

```php
class UserService{
  protected $blogService;
  
  public function __construct(BlogService $blogService){
    $this->blogService = $blogService;
  }
  
  public function create(array $data) : User {
    $user = new User;
    $user->email = $data['email'];
    $user->password = $data['password'];
    $user->save();
    
    $blog = $this->blogService->create();
    $user->blogs()-attach($blog);
    
    return $user;
  }
  
  ...
}
```

这样看上去就好多了；我们可以直接从 Controller 里调用 User 的 `create()`/ `delete()` ，通过任意途径返回结果。然而这方式有个问题：*我们处理业务逻辑的时候很少使用一种模式*。

例如我们有这个业务逻辑：当用户创建账号的时候，需要创建一篇新的博客。如果我们继续使用刚才的模式，我们需要创建 `BlogService` 然后使其作为依赖注入到 `UserService` 中：

```php
class UserSerivce{
  public function create(array $data) : User {
    $user = new User;
    $user->email = $data['email'];
    $user->password = $data['password'];
    $user->save();
    
    return $user;
  }
  
  public function delete($userId) : bool {
    $user = User::findOrFail($userId);
    $user->delete();
    
    return true;
  }
}
```

很显然，当我们的应用变得越来越大，会有越来越多的服务类，有些还依赖了 5 个甚至 6 个其他的服务类，代码像被猫玩过的毛线球剪不断理还乱 —— 我们无论如何都要避免这种结局。

## 引入动作类

如果我们不使用里面塞了几个方法的单一服务类，而是把逻辑切分到好几个类里面呢？我在最近的好几个工程里都使用了这种模式，结果十分不错。

首先，我们暂时先把“服务”这个抽象而且模糊的术语丢掉，然后引入“动作”类，并且赋予以下定义：

- 一个动作类，它的名字应该要能够让人一眼看出它是干什么的，例如：*CreateOrder, ConfirmCheckout, DeleteProduct, AddProductToCart,…..*
- 动作类应该只拥有一个公开接口（API）。理想情况下应该用统一的名字，例如 *handle()*, *execute()* 这样，方便我们需要在动作上实现一些模式，例如适配器。
- 动作类不应该关心 *Request* 和 *Response*。动作类不处理任何 *Request*，也不产生任何 *Response*，因为这是 *Controller* 的责任。
- 动作类可以依赖其他的动作。
- 当处理业务逻辑的时候，遇到不能返回期望内容的情况下，必须抛出异常，让调用者（或者 Laravel 的 *ExceptionHandler*）去负责渲染或者返回错误信息。

## 创建 CreateUser 动作

现在，让我们用 *CreateUser* 动作类来重构刚才的例子。

```php
class CreateUser {
  protected $createBlog;
  
  public function __constructure(CreateBlog $createBlog){
    $this->createBlog = $createBlog;
  }
  
  public function execute(array $data) : User {
    $email = $data['email'];
    
    if(User::whereEmail($email)->first()) {
      throw new EmailNotUniqueException("$email shoould be unique.");
    }
    
    $user = new User;
    $user->email = $data['email'];
    $user->password = $data['password'];
    $user->save();
    
    $blog = $this->createBlog->execute();
    $user->blogs()->attach($blog);
    
    return $user;
  }
}
```


你可能觉得奇怪，为什么代码要在 Email 存在的情况下抛出异常，这个不应该是在请求的时候就应该验证一次吗？当然应该让验证器去做这件事，不过，强制动作类遵循业务逻辑是一个好主意。这样做不但能够让业务逻辑更明确（不用过多考虑异常情况），也更容易调试。

下面是我们使用动作类重构后的控制器：

```php
class UserController{
  public function create(CreateUserRequest $request, CreateUser $action){
    $user = $action->execute($request->all());
    
    return view('user.created',['user' => $user]);
 }
}

class UserApiController{
  public function create(CreateUserRequest $request, CreateUser $action){
    $user = $action->execute($request->all());
    
    return response()->json($user);
  }
}
```


现在我们再也不用在注册流程更改的时候同时修改两边的代码，干净整洁。

## 内嵌动作

当我们需要往应用里倒入 1000 个用户的时候，我们可以写一个依赖 *CreateUser* 的动作类：

```php
class ImportUsers{
  protected $createUser;
  
  public function __construct(CreateUser $createUser){
    $this->createUser = $createUser;
  }
  
  public function execute(array $rows) : Collection{
    return collect($row)->map(function(array $row){
      try{
        return $this->createUser->execute($row);
      } catch(EmailNotUniqueException $e){
        // Deal with duplicate users
      }
    });
  }
}
```

干净整洁，简单易懂。在这里我们在 `Collection::map()` 里重用了 *CreateUser* ，返回一个集合，装着新鲜出炉的用户们。我们可以稍作优化：当有重复邮件的时候，我们可以通过返回 *Null* 对象，或者往 *Logger* 里丢 INFO ，随你喜欢。

# 装饰一下 Actions

现在，假设我们要往日志里记录每一个用户的注册。我们可以直接把代码放到动作自身上，但我们也可以使用**装饰器模式**。

```php
class LogCreateUser extends CreateUser{
  public function execute(array $data) : User {
  	Log::info("A new user has registered : ". $data['email']);
    
    return parent::execute($data);
  }
}
```

然后，使用 Laravel 的 IoC 容器，我们可以把 *LogCreateUser* 类绑在 *Createuser* 类上，这样当我们就总能在需要后者的时候把原型注入进去：

```php
public function register(){
  $this->app->bind(CreateUser::class, LogCreateUser::class);
}
```

这样做甚至更容易地通过环境变量或者配置控制日志的开启和关闭：

```php
public function register(){
  if(config("user.log_registeration")) {
    $this->app->bind(CreateUser::class, LogCreateUser::class);
  }
}
```

## 总结

使用这种模式大概会在开始先产生一大堆的类。当然，为了减少一下文章篇幅读起来比较方便，这里只举了这个简单的用户注册例子。当复杂度增加的时候，这个模式的价值就开始体现出来了 — —  因为你清楚这些边界清楚分工明确的代码在哪。

### 使用动作类的好处：

- 使用小物件管理领域逻辑可避免重复代码以及增加可重用性，Keep things SOLID。
- 易于针对不同的场景做独立测试。
- 清晰易懂、有意义的命名让理解项目变得更加容易
- 跨项目一致性：避免代码七零八落地分布在 *Controller*、*Models* 等等……

而且，这些实践都是基于我最近这些年使用 Laravel 的经验以及我写过的项目总结出来的。它们对我来说非常管用，以至于我甚至在中小型项目里使用。

我非常希望能够与你一同交流分享，如果你有不同的实践方式那就最好不过了！