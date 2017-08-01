# Phalcon 分享

## 一、前言

### 1.1 概念

> [参考文章](http://www.digpage.com/di.html#)

#### 1.1.1 Container （容器）

* 管理对象的生成、资源取得、销毁等生命周期
* 建立对象与对象之间的依赖关系
* 启动容器后，所有对象直接取用，不用编写任何一行代码来产生对象，或是建立对象之间的依赖关系。

#### 1.1.2 依赖倒置原则（Dependence Inversion Principle, DIP）

> DIP是一种软件设计的指导思想。传统软件设计中，上层代码依赖于下层代码，当下层出现变动时， 上层代码也要相应变化，维护成本较高。而DIP的核心思想是上层定义接口，下层实现这个接口， 从而使得下层依赖于上层，降低耦合度，提高整个系统的弹性。这是一种经实践证明的有效策略。

#### 1.1.3 控制反转（Inversion of Control, IoC）

> IoC就是DIP的一种具体思路，DIP只是一种理念、思想，而IoC是一种实现DIP的方法。 IoC的核心是将类（上层）所依赖的单元（下层）的实例化过程交由第三方来实现。 一个简单的特征，就是类中不对所依赖的单元有诸如 `$component = new yii\component\SomeClass（）` 的实例化语句。

#### 1.1.4 依赖注入（Dependence Injection, DI）

> DI是IoC的一种设计模式，是一种套路，按照DI的套路，就可以实现IoC，就能符合DIP原则。 DI的核心是把类所依赖的单元的实例化过程，放到类的外面去实现。

#### 1.1.5 控制反转容器（IoC Container）

> 当项目比较大时，依赖关系可能会很复杂。 而IoC Container提供了动态地创建、注入依赖单元，映射依赖关系等功能，减少了许多代码量。 Yii 设计了一个 yii\di\Container 来实现了 DI Container。

#### 1.1.6 服务定位器（Service Locator）

> Service Locator是IoC的另一种实现方式， 其核心是把所有可能用到的依赖单元交由Service Locator进行实例化和创建、配置， 把类对依赖单元的依赖，转换成类对Service Locator的依赖。 DI 与 Service Locator并不冲突，两者可以结合使用。 目前，Yii2.0把这DI和Service Locator这两个东西结合起来使用，或者说通过DI容器，实现了Service Locator。

### 1.2 IoC实例

* [控制反转 Inversion of Control](https://zh.wikipedia.org/wiki/%E6%8E%A7%E5%88%B6%E5%8F%8D%E8%BD%AC)
* 依赖关系的转移
* 依赖抽象而非实践

假设应用程序有储存需求，若直接在高层的应用程序中调用低层模块API，导致应用程序对低层模块产生依赖。

```php
/**
 * 高层
 */
class Business
{
    private $writer;

    public function __construct()
    {
        $this->writer = new FloppyWriter();
    }

    public function save()
    {
        $this->writer->saveToFloppy();
    }
}

/**
 * 低层，软盘存储
 */
class FloppyWriter
{
    public function saveToFloppy()
    {
        echo __METHOD__;
    }
}

$biz = new Business();
$biz->save(); // FloppyWriter::saveToFloppy
```

假设程序要移植到另一个平台，而该平台使用USB磁盘作为存储介质，则这个程序无法直接重用，必须加以修改才行。本例由于低层变化导致高层也跟着变化，不好的设计。

正如1.1.2提到的，程序不应该依赖于具体的实现，而是要依赖抽像的接口。

```php
/**
 * 接口
 */
interface IDeviceWriter
{
    public function saveToDevice();
}

/**
 * 高层
 */
class Business
{
    /**
     * @var IDeviceWriter
     */
    private $writer;

    /**
     * @param IDeviceWriter $writer
     */
    public function setWriter($writer)
    {
        $this->writer = $writer;
    }

    public function save()
    {
        $this->writer->saveToDevice();
    }
}

/**
 * 低层，软盘存储
 */
class FloppyWriter implements IDeviceWriter
{

    public function saveToDevice()
    {
        echo __METHOD__;
    }
}

/**
 * 低层，USB盘存储
 */
class UsbDiskWriter implements IDeviceWriter
{

    public function saveToDevice()
    {
        echo __METHOD__;
    }
}

$biz = new Business();
$biz->setWriter(new UsbDiskWriter());
$biz->save(); // UsbDiskWriter::saveToDevice

$biz->setWriter(new FloppyWriter());
$biz->save(); // FloppyWriter::saveToDevice
```

控制权从实际的FloppyWriter转移到了抽象的IDeviceWriter接口上，让Business依赖于IDeviceWriter接口，且FloppyWriter、UsbDiskWriter也依赖于IDeviceWriter接口。

这就是IoC，面对变化，高层不用修改一行代码，不再依赖低层，而是依赖注入，这就引出了DI。

比较实用的注入方式有三种：

* Setter injection 使用setter方法
* Constructor injection 使用构造函数
* Property Injection 直接设置属性


事实上不管有多少种方法，都是IoC思想的实现而已，上面的代码演示的是Setter方式的注入。

### 1.3 依赖注入容器


* 依赖注入 Dependency Injection
* 不必自己在代码中维护对象的依赖
* 容器自动根据配置，将依赖注入指定对象
* 管理应用程序中的『全局』对象（包括实例化、处理依赖关系）。
* 可以延时加载对象（仅用到时才创建对象）。
* 促进编写可重用、可测试和松耦合的代码。

理解了IoC和DI之后，就引发了另一个问题，引用Phalcon文档描述如下：

如果这个组件有很多依赖， 我们需要创建多个参数的setter方法​​来传递依赖关系，或者建立一个多个参数的构造函数来传递它们，另外在使用组件前还要每次都创建依赖，这让我们的代码像这样不易维护

```php
//创建依赖实例或从注册表中查找
$connection = new Connection();
$session = new Session();
$fileSystem = new FileSystem();
$filter = new Filter();
$selector = new Selector();

//把实例作为参数传递给构造函数
$some = new SomeComponent($connection, $session, $fileSystem, $filter, $selector);

// ... 或者使用setter

$some->setConnection($connection);
$some->setSession($session);
$some->setFileSystem($fileSystem);
$some->setFilter($filter);
$some->setSelector($selector);
```

假设我们必须在应用的不同地方使用和创建这些对象。如果当你永远不需要任何依赖实例时，你需要去删掉构造函数的参数，或者去删掉注入的setter。为了解决这样的问题，我们再次回到全局注册表创建组件。不管怎么样，在创建对象之前，它增加了一个新的抽象层：

```php
class SomeComponent
{

    // ...

    /**
     * Define a factory method to create SomeComponent instances injecting its dependencies
     */
    public static function factory()
    {

        $connection = new Connection();
        $session = new Session();
        $fileSystem = new FileSystem();
        $filter = new Filter();
        $selector = new Selector();

        return new self($connection, $session, $fileSystem, $filter, $selector);
    }

}
```

瞬间，我们又回到刚刚开始的问题了，我们再次创建依赖实例在组件内部！我们可以继续前进，找出一个每次能奏效的方法去解决这个问题。但似乎一次又一次，我们又回到了不实用的例子中。

一个实用和优雅的解决方法，是为依赖实例提供一个容器。这个容器担任全局的注册表，就像我们刚才看到的那样。使用依赖实例的容器作为一个桥梁来获取依赖实例，使我们能够降低我们的组件的复杂性：

```php
class SomeComponent
{

    protected $_di;

    public function __construct($di)
    {
        $this->_di = $di;
    }

    public function someDbTask()
    {

        // 获得数据库连接实例
        // 总是返回一个新的连接
        $connection = $this->_di->get('db');

    }

    public function someOtherDbTask()
    {

        // 获得共享连接实例
        // 每次请求都返回相同的连接实例
        $connection = $this->_di->getShared('db');

        // 这个方法也需要一个输入过滤的依赖服务
        $filter = $this->_di->get('filter');

    }

}

$di = new Phalcon\DI();

//在容器中注册一个db服务
$di->set('db', function() {
    return new Connection(array(
        "host" => "localhost",
        "username" => "root",
        "password" => "secret",
        "dbname" => "invo"
    ));
});

//在容器中注册一个filter服务
$di->set('filter', function() {
    return new Filter();
});

//在容器中注册一个session服务
$di->set('session', function() {
    return new Session();
});

//把传递服务的容器作为唯一参数传递给组件
$some = new SomeComponent($di);

$some->someTask();
```

这个组件现在可以很简单的获取到它所需要的服务，服务采用延迟加载的方式，只有在需要使用的时候才初始化，这也节省了服务器资源。这个组件现在是高度解耦。例如，我们可以替换掉创建连接的方式，它们的行为或它们的任何其他方面，也不会影响该组件。

### 1.4 参考文章

* [PHP程序员如何理解依赖注入容器(dependency injection container)](https://segmentfault.com/a/1190000002424023)
* [依赖注入与服务定位器](http://www.iphalcon.cn/reference/di.html)
* [What is Dependency Injection? Fabien Potencier](http://fabien.potencier.org/article/11/what-is-dependency-injection)
* [Inversion of Control Containers and the Dependency Injection pattern](http://martinfowler.com/articles/injection.html) by Martin Fowler


## 二、Phalcon简介

### 2.1 声明
    
目前最新版本：3.2，本次分享基于[文档3.1.1](http://www.iphalcon.cn/index.html)

### 2.2 基本功能

#### 2.2.1 低开销

    低内存消耗和 CPU 相比传统的框架
    
* Zephir/C 扩展的加载与 PHP web 服务器守护进程启动进程一次
* 类和函数提供的扩展都准备好要使用的任何应用程序
* 代码编译并不解释，因为它已经被编译为一个特定的平台和处理器
* 由于其低层建筑和 **Phalcon提供基于 MVC 的应用程序的最低开销** 的优化
    
#### 2.2.2 MVC&HMVC：模块、 组件、 模型、 视图和控制器

与轻松和快乐建立单一和多模块的应用程序。你已经知道使用文件结构、 方案和模式。
    
```
single/
app/
   controllers/
   models/
   views/
public/
   css/
   img/
   js/
```
    
```
multiple/
apps/
  frontend/
     controllers/
     models/
     views/
     Module.php
  backend/
     controllers/
     models/
     views/
     Module.php
  public/
  ../
```
    
#### 2.2.3 依赖注入：依赖注入和位置的服务和它的本身他们的容器。

Phalcon 是建成后一个功能强大但易于理解和使用模式被称为依赖注入。初始化或定义服务一次-和几乎任何地方使用它们整个应用程序。
    
```php
// 创建依赖项注入器容器
$di = new Phalcon\DI();
    
//注册类、 函数、 组件
$di->set("request", new Phalcon\Http\Request());
    
..
    
// 在代码中的其他任何地方使用
$request = $di->getShared('request');
```

#### 2.2.4 Rest

写作其余服务器和应用程序从未如此简单。没有样板。简单的服务将适合在一个文件中。
    
```php
use Phalcon\Mvc\Micro;
    
$app = new Micro();
    
// 返回 json 格式的数据
$app->get(
   '/check/status',
   function () {
       return $this->response->setJsonContent(
           [
               'status' => 'important',
           ]
       );
   }
);
    
$app->handle();
```

#### 2.2.5 自动加载：提供符合PSR-4标准的自动加载机制

注册命名空间、 前缀、 目录或类。利用自动加载事件和保持充分控制哪些文件被加载并从何处。
    
```php
use Phalcon\Loader;

// 创建自动加载
$loader = new Loader();
    
// 注册一些命名空间
$loader->registerNamespaces(
   [
      'Example\Base'    => 'vendor/example/base/',
      'Example\Adapter' => 'vendor/example/adapter/',
      'Example'         => 'vendor/example/',
   ]
);
    
// 注册自动加载
$loader->register();
```

#### 2.2.6 路由器：Phalcon\Mvc\Router 提供了先进的路由功能。

它应该是路由。没什么更多。没有什么更少。
    
```php
// 创建路由器
$router = new \Phalcon\Mvc\Router();
    
// 定义路由
$router->add(
  '/admin/users/my-profile',
  [
      'controller' => 'users',
      'action'     => 'profile',
  ]
);
```

### 2.3 数据存储

#### 2.3.1 ORM：对象关系映射

### 2.4 前端

#### 2.4.1 模板引擎

发表的意见代表您的应用程序的用户界面。观点往往与嵌入 PHP 代码执行任务仅涉及数据的演示文稿的 HTML 文件。意见处理提供到 web 浏览器或其他工具，用来使您的应用程序的请求数据的工作
    
```html
<html>
   <body>
   <div class='top'><?php $this->partial('shared/ad_banner'); ?></div>
   <div class='content'>
       <h1>Robots</h1>
       <p>Check out our specials for robots:</p>
       ...
   </div>
   <div class='footer'><?php $this->partial('shared/footer'); ?></div>
   </body>
</html>    
```                

### 2.5 更多


## 三、 hello world

### 3.1 目录结构

Phalcon不会强制要求应用程序的开发遵循特定的文件结构。因为它是松散耦合的，你可以实现Phalcon驱动的应用程序，以及使用对你来说最舒服的文件结构。

建议采用以下格式

```
tutorial/
  app/
    controllers/
    models/
    views/
  public/
    css/
    img/
    js/
```

不需要任何有关Phalcon的 “library” 目录。该框架已经被加载到内存中

### 3.2 引导程序

1.  设置自动加载器.

2.  配置依赖注入.

3.  处理应用请求.

### 3.3 自动加载

[Phalcon\\Loader](http://www.iphalcon.cn/api/Phalcon_Loader.html) **组件**

-   注册一个自动加载器，用于加载控制器和模型类
-   可以为控制器注册一个或多个目录来增加应用程序的灵活性的
-   可以加载使用各种策略类

在这个例子中，我们选择了在预定义的目录中查找类

```php
$loader = new Loader();

$loader->registerDirs(
    [
        "../app/controllers/",
        "../app/models/",
    ]
);

$loader->register();
```

### 3.4 依赖管理

#### 3.4.1  [依赖注入容器](http://www.iphalcon.cn/reference/di.html)

服务容器是一个全局存储的将要被使用的应用程序功能包。每次框架需要的一个组件时，会请求这个使用协定好名称的服务容器。因为Phalcon是一个高度解耦的框架，
Phalcon\Di
作为黏合剂，促使不同组件的集成，以一个透明的方式实现他们一起进行工作。

```php
use Phalcon\Di\FactoryDefault;

// ...

// 创建一个 DI
$di = new FactoryDefault();
```

[Phalcon\\Di\\FactoryDefault](http://www.iphalcon.cn/api/Phalcon_Di_FactoryDefault.html) 是 [Phalcon\\Di](http://www.iphalcon.cn/api/Phalcon_Di.html) 的一个变体。为了让事情变得更容易，它已注册了Phalcon的大多数组件。
因此，我们不需要一个一个注册这些组件。在以后更换工厂服务的时候也不会有什么问题。

#### 3.4.2  注册视图服务

在接下来的部分，我们注册了“视图(view)”服务，指示框架将去指定的目录寻找视图文件。由于视图并非PHP类，它们不能被自动加载器加载。

服务可以通过多种方式进行登记，但在我们的教程中，我们将使用一个匿名函数 [anonymous function](http://php.net/manual/zh/functions.anonymous.php):

```php
use Phalcon\Mvc\View;

// ...

// 设置视图组件
$di->set(
  "view",
  function () {
      $view = new View();

      $view->setViewsDir("../app/views/");

      return $view;
  }
);
```

#### 3.4.3  注册基础URI

接下来，我们注册一个基础URI，这样通过Phalcon生成包括我们之前设置的“tutorial”文件夹在内的所有的URI。
我们使用类 Phalcon\Tag 生成超链接，这将在本教程后续部分很重要。

```php
use Phalcon\Mvc\Url as UrlProvider;

// ...

// 设置一个基础URI, 这样所有生成的URI都包含"tutorial"文件夹
$di->set(
  "url",
  function () {
      $url = new UrlProvider();

      $url->setBaseUri("/tutorial/");

      return $url;
  }
);
```

### 3.5 处理应用请求

在这个文件的最后部分，我们发现 [Phalcon\\Mvc\\Application](http://www.iphalcon.cn/api/Phalcon_Mvc_Application.html)。其目的是初始化请求环境，并接收路由到来的请求，接着分发任何发现的动作；收集所有的响应，并在过程完成后返回它们。

```php
use Phalcon\Mvc\Application;

// ...

$application = new Application($di);

$response = $application->handle();

$response->send();
```

### 3.6 创建控制器

默认情况下Phalcon会寻找一个名为“Index”的控制器。当请求中没有控制器或动作时，则使用“Index”控制器作为起点。这个“Index”控制器
(app/controllers/IndexController.php) 看起来类似：

```php
use Phalcon\Mvc\Controller;

class IndexController extends Controller
{
    public function indexAction()
    {
        echo "<h1>Hello!</h1>";
    }
}
```

注意：**该控制器类必须有“Controller”后缀，且控制器动作必须有“Action”后缀。**

### 3.7 输出到视图

使用类 [Phalcon\\Tag](http://www.iphalcon.cn/api/Phalcon_Tag.html) 去生成标记。
这是一个让我们构建HTML标记的实用类。
关于生成HTML更详细的文章可以查看 [视图助手](http://www.iphalcon.cn/reference/tags.html)

#### 3.7.1  创建view

```php
echo "<h1>Hello!</h1>";

echo PHP_EOL;

echo PHP_EOL;

echo $this->tag->linkTo(
  "signup",
  "Sign Up Here!"
);
```

上述代码等同于以下代码

```html
<h1>Hello!</h1>

<a href="/tutorial/signup">Sign Up Here!</a>
```

#### 3.7.2  创建signup/index.phtml

```html
<h2>
  Sign up using this form
</h2>

<?php echo $this->tag->form("signup/register"); ?>

  <p>
      <label for="name">
          Name
      </label>

      <?php echo $this->tag->textField("name"); ?>
  </p>

  <p>
      <label for="email">
          E-Mail
      </label>

      <?php echo $this->tag->textField("email"); ?>
  </p>



  <p>
      <?php echo $this->tag->submitButton("Register"); ?>
  </p>

</form>
```

[Phalcon\\Tag](http://www.iphalcon.cn/api/Phalcon_Tag.html) 还提供了有用的方法来构建表单元素。

Phalcon\Tag::form()方法只接受一个参数实例,
一个相对uri到这个应用的一个控制器/动作。

通过单击“Send”按钮，您将注意到框架抛出了一个异常，这表明我们是错过了在控制器中注册“register”动作。我们的
public/index.php 文件抛出这个异常：

```
Exception: Action “register” was not found on handler “signup”
```

###3 .8 创建模型

#### 3.8.1  sql

    ```sql
    CREATE TABLE `users` (
       `id`    int(10)     unsigned NOT NULL AUTO_INCREMENT,
       `name`  varchar(70)          NOT NULL,
       `email` varchar(70)          NOT NULL,

       PRIMARY KEY (`id`)
    );
    ```

#### 3.8.2  models/User.php 模型

    ```php
    use Phalcon\Mvc\Model;

    class Users extends Model
    {
       public $id;

       public $name;

       public $email;
    }
    ```

### 3.9 设置数据库连接

```php
use Phalcon\Db\Adapter\Pdo\Mysql as DbAdapter;

// 设置数据库服务
$di->set(
    "db",
    function () {
        return new DbAdapter(
            [
                "host"     => "localhost",
                "username" => "root",
                "password" => "mysql",
                "dbname"   => "test",
            ]
        );
    }
);
```

### 3.10 使用模型保存数据

registerAction方法：

```php
public function registerAction()
{
   // 实例一个用户类
   $user = new Users();

   // 存储和检验错误
   $success = $user->save(
       $this->request->getPost(),
       [
           "name",
           "email",
       ]
   );

   if ($success) {
       echo "Thanks for registering!";
   } else {
       echo "Sorry, the following problems were generated: ";

       $messages = $user->getMessages();

       foreach ($messages as $message) {
           echo $message->getMessage(), "<br/>";
       }
   }

   $this->view->disable();
}
```

类的公共属性映射到用户表中的记录的字段。
在新记录中设置相应的值并调用:code:`save()`将在数据库中存储的数据记录。:code:`save()`方法返回一个布尔值，表示存储的数据是否成功。

ORM自动转义输入以防止SQL注入，所以我们只需要将请求传递给:code:`save()`方法。   


## 四、 加载目录结构

> 关于单项目与多项目目录机构，请参考2.2.2

> 本部分内容属于[类加载器](http://www.iphalcon.cn/reference/loader.html)，不展开分享，仅仅分享关于目录结构的内容。

### 4.1 注册命名空间

如果你的代码用命名空间组织，或者你要使用的外部类库使用了命名空间，那么 `registerNamespaces()` 方法提供了相应的加载机制。它接收一个关联数组作为参数，键名是命名空间的前缀，值是这些类对应的文件所在的目录。 当加载器尝试寻找文件时，命名空间的分隔符会被替换成目录分隔符。记得在路径的末尾加上斜杠。

```php
use Phalcon\Loader;

// Creates the autoloader
$loader = new Loader();

// Register some namespaces
$loader->registerNamespaces(
    [
       "Example\Base"    => "vendor/example/base/",
       "Example\Adapter" => "vendor/example/adapter/",
       "Example"         => "vendor/example/",
    ]
);

// Register autoloader
$loader->register();

// The required class will automatically include the
// file vendor/example/adapter/Some.php
$some = new \Example\Adapter\Some();
```

### 4.2 注册文件夹

另一个方式是注册存放类文件的文件夹。由于**性能问题这个方式并不推荐**，因为Phalcon在目录里面查找跟类名同名的文件的时候，会在每个目录里面产生相当多的 file stats 操作。 请注意按照相关的顺序注册文件夹，同时，记得在每个文件夹路径末尾加上斜杠。

```php
use Phalcon\Loader;

// Creates the autoloader
$loader = new Loader();

// Register some directories
$loader->registerDirs(
    [
        "library/MyComponent/",
        "library/OtherComponent/Other/",
        "vendor/example/adapters/",
        "vendor/example/",
    ]
);

// Register autoloader
$loader->register();

// The required class will automatically include the file from
// the first directory where it has been located
// i.e. library/OtherComponent/Other/Some.php
$some = new \Some();
```

### 4.3 注意事项

在使用加载器的时候，一些关键点需要牢记于心：

* 自动加载处理过程是大小写敏感的。类文件名与代码中所写的一致。
* 基于namespaces/prefixes机制的加载策略比基于directories的要快。
* 如果安装了类似 APC 的缓存工具，加载器隐式的用它来缓存文件检索结果，以便提高性能。

### 4.4 总结

就像文档中说的“Phalcon不会强求应用程序使用特定的文件结构”，我们可以根据业务需要，或者开发习惯，自由组织我们的项目目录结构。只需要通过"[Autoloader](http://www.iphalcon.cn/reference/loader.html)"注册这些目录结构，即可正常使用。

## 五、控制器

在一个项目中，没有强制指定放置控制器的地方，这些控制器都可以 通过使用 autoloaders 来加载，所以你可以根据需要自由组件你的控制器。

### 5.1 命名规则

控制器类必须以“Controller”为后缀，action则须以“Action”为后缀。一个控制器类的例子如下：

```php
use Phalcon\Mvc\Controller;

class PostsController extends Controller
{
    public function indexAction()
    {

    }

    public function showAction($year, $postTitle)
    {

    }
}
```

### 5.2 初始化

* onConstruct:紧接着创建控制器对象的后面执行一些初始化的逻辑

* initialize:初始化的函数，它会最先执行，并优于任何控制器的其他action。

#### 5.2.1 initialize()

Phalcon\Mvc\Controller 中提供了初始化函数 `initialize()`，它是最先执行的，并且会优先于任何控制器的其他action：

#### 5.2.2 onConstruct()

Phalcon\Mvc\Controller 控制器基类中，`__construct()` 函数已经被声明为 final ，明确禁止子类重写此函数，因此提供了另一个方法 `onConstruct()`来执行初始化的逻辑

#### 5.2.3 对比

进行如下测试：

```php
public function initialize() {
   var_dump('initialize');

   echo '<pre>';
   print_r($this);
}

public function onConstruct(){
   var_dump('onConstruct');

   echo '<pre>';
   print_r($this);
}

public function indexAction(){
   echo 'front index';
   exit();
}
```

测试结果区别：

1. 执行顺序：先 onConstruct() -> initialize()
2. onConstruct() 是实例化对象的过程，相当于 new
3. initialize() 是初始化资源的过程。如加载DI中注册的所有服务


### 5.3 接收请求数据

#### 5.3.1 pathinfo方式

    url : ...Controller/index/phalcon/passphalcon

```php
public function indexAction($Username, $Passwd, $email = 'cw490@126.com')
{
   echo $Username . '</br>';
   echo $Passwd . '</br>';
   echo $email;
   echo '<h1>Controller/index!</h1>';
   $this->view->disable();
}
```

**注意:当请求是没有传递参数1和参数2则会引起报错**

#### 5.3.2 普通方式

```php
 //  http://localhost/index/test1?a=1&b=2
 public function test1Action(){
   $a = $this->request->get('a');
   $b = $this->request->getQuery('b');
   var_dump("a:{$a}");
   var_dump("b:{$b}");
   exit;
}

```

注意：

* $this->request->get() 方法能同时获取 GET 和 POST 请求的数据；
* $this->request->getQuery() 只能获取 GET 方式的请求数据；
* $this->request->getPost() 只能获取 POST 方式的请求数据。

#### 5.3.3 通过`$dispatcher`接收

    dispatcher调度控制器，对 router或者特定的url参数给与重组

```php
 public function test3Action(){
   $a = $this->dispatcher->getParam('a');
   $b = $this->dispatcher->getParam('b');
   var_dump($a);
   var_dump($b);
}
```

路由配置规则：

```php
'/index/test3/(\d+)/(\d+)' => array(
   'module' => 'frontend',
   'controller'=>'index',
   'action'=>'test3',
   'a' => 1,
   'b' => 2,
),
```

在浏览器中访问 http://localhost/index/test3/111/222 即可看到打印的结果。

### 5.4 返回响应数据

```php
public function test6Action(){
   return $this->response->setJsonContent(array(
       'code' => 1,
       'message' => 'success',
   ));
}
```

在浏览器中访问 http://localhost/index/test6 即可看到ajax返回的JSON数据。

### 5.5 页面跳转与转发

Phalcon中提供了两种页面跳转方式。

#### 5.5.1 redirect() 跳转

```php
public function test4Action(){
   return $this->response->redirect('https://www.marser.cn');
}
```

**浏览器中的URL地址已经发生了变化。**

#### 5.5.2 forward() 转发

说道转发可能有这样一个场景,一个管理员用户请求了过来但是这个用户并没有使用这个业务的权限,我们需要让用户看到无权限提示,其实在前面介绍返回的时候已经可以使用返回的重定向跳转到无权限提示页面,或者可以使用如下方式:

```php
public function test5Action(){
   return $this->dispatcher->forward(array(
       'controller' => 'test',
       'action' => 'index',
   ));
}
```

* 此种方式的页面跳转不会改变URL地址，只是将请求转发到另一个控制器的action。
* 通过转发之后本方法内的代码依然会被执行建议在转发之后直接return不然后面的代码会继续执行


### 5.6 调用DI中注册的服务

    使用session举例

#### 5.6.1 初始化session

在index.php中添加如下代码，初始化session
    
```php

// 在一个组件请求Session服务的时候, 启动Sesssion
$di->set(
    "session",
    function () {
        $session = new Session();

        $session->start();

        return $session;
    }
);
```

#### 5.6.2 使用session

```php
public function index3Action() {

   // 以和服务相同名字的类属性访问
   $this->session->set('phalcon', 'test');                            
   // 另一种方式：使用魔法getter来访问
   echo $this->di->getsession()->get('phalcon') . '</br>';            
   // 通过DI访问服务
   echo $this->di->get('session')->get('phalcon') . '</br>';          
   // 使用数组下标
   echo $this->di['session']->get('phalcon') . '</br>';               
   // 通过getDI方法获取实例
   echo $this->getDI()->getsession()->get('phalcon') . '</br>';       

   echo '<h1>Controller/index3!</h1>';
}
```

更多session，请查看[使用 Session 存储数据](http://www.iphalcon.cn/reference/session.html)

## 六、 视图

    volt是Phalcon中集成的模板引擎，我们也可以更换为其他模板引擎或同时使用多个模板引擎。

### 6.1 启用Volt

我们需要将 volt 模板注册到 views 组件中，并设置模板文件通用后缀名，或者直接使用标准化的后缀名 .phtml 才能正常使用

```php
$di->setShared('view', function () use ($config, $di) {
    $view = new \Phalcon\Mvc\View();
    //设置模板根目录
    $view->setViewsDir(ROOT_PATH . '/app/views/');
    //注册模板引擎
    $view->registerEngines(array(
        //设置模板后缀名
        '.phtml' => function ($view, $di) use ($config) {
            $volt = new \Phalcon\Mvc\View\Engine\Volt($view, $di);
            $volt->setOptions(array(
                //模板是否实时编译
                'compileAlways' => false,
                //模板编译目录
                'compiledPath' => ROOT_PATH . '/app/cache/compiled/frontend'
            ));
            return $volt;
        },
    ));
    return $view;
});
```

### 6.2 原理

* 把 action 中传过来的数据定义成变量。
* 加载视图文件，此时视图里的变量已经有值了 （如果是volt的话，把volt语法编译成原生php，加载编译后的文件），它是一个 php 和 html 混写的文件，php 的代码会执行，最后只剩下 html。
* 把执行完后的内容返回出去。

### 6.3 控制器传值给视图

使用方法：

```php
// 直接设置变量，会调用 __set() 的魔术方法
$this->view->products = $products;

// setParamToView()
$this->view->setParamToView('products', $products);

// setVar()
$this->view->setVar('products', $products);

// setVars()
$this->view->setVars(array('products' => $products));
```

这几个方法都是把值设置给了 view 实例的一个属性 _viewParams ，在调用 _engineRender() 方法时，它取出 _viewParams 的值作为参数传给了具体的渲染引擎。

### 6.4 用法

`volt` 模板中 **基本用法** 、 **变量** 、**表达式** 、 **流程控制**等部分的具体用法，文档中已有详细说明，请直接翻阅[Phalcon文档](http://www.iphalcon.cn/reference/volt.html#basic-usage) 。这里分享一下大家咨询比较多的几处用法以及踩过的坑。

#### 6.4.1 控制器指定模板

```php
 public function testAction(){
   $this->view->pick('view/test');
}
```

#### 6.4.2 变量传值

```php
//控制器中变量
public function test2Action(){
   //setVar：单独进行变量传值
   $this->view->setVar('test', 'hello world');

   //setVars：关联数组进行变量传值
   //$this->view->setVars([
   //    'test' => 'hellow world',
   //]);
   $this->view->pick('view/test2');
}
```

#### 6.4.3 数值循环（For）

Phalcon文档 中有提到 volt 模板中 对象 和 关联数组 的循环方式，但没有明确例子来说明数值循环的用法。具体用法请看如下示例代码：

```php
{% for i in 0..100 %}
  <div>{{i}}</div>
{% endfor %}
```

#### 6.4.4 连接符

在 volt 模板中的连接符不是 . ，也不是 + ，而是 ~，代码示例如下：

```php
{{ url('user/detail?uid='~user['uid']) }}
```

#### 6.4.5 选择视图

视图渲染的是最后的一个相关的控制器和执行动作。你可以使用 Phalcon\Mvc\View::pick() 方法覆盖它

```php
<?php

use Phalcon\Mvc\Controller;

class ProductsController extends Controller
{
    public function listAction()
    {
        // Pick "views-dir/products/search" as view to render
        $this->view->pick("products/search");

        // Pick "views-dir/books/list" as view to render
        $this->view->pick(
            [
                "books",
            ]
        );

        // Pick "views-dir/products/search" as view to render
        $this->view->pick(
            [
                1 => "search",
            ]
        );
    }
}
```

#### 6.4.6 关闭视图

如果你的控制器不在视图里产生(或没有)任何输出，你可以禁用视图组件来避免不必要的处理，或者采用return方式坐处理

```php
use Phalcon\Mvc\Controller;

class UsersController extends Controller
{
    public function closeSessionAction()
    {
        // Close session
        // ...

        // Disable the view to avoid rendering
        $this->view->disable();
    }
}
```

#### 6.4.7 局部模版

局部模板是把渲染过程分解成更简单、更好管理的、可以重用不同部分的应用程序块的另一种方式。

```html
<div class="top"><?php $this->partial("shared/ad_banner"); ?></div>

<div class="content">
    <h1>Robots</h1>

    <p>Check out our specials for robots:</p>
    ...
</div>

<div class="footer"><?php $this->partial("shared/footer"); ?></div>
```

方法 partial() 也接受一个只存在于局部范围的变量/参数的数组作为第二个参数:

```html
<?php $this->partial("shared/ad_banner", ["id" => $site->id, "size" => "big"]); ?>
```

### 6.4.8 渲染级别

* [Phalcon View 多个渲染级别之间的关系](Phalcon View 多个渲染级别之间的关系)
* [Phalcon View 渲染原理及过程](https://segmentfault.com/a/1190000004358686)

## 七、视图助手

因为HTML标签的命名方式和很多标签属性，让书写HTML标签变成一项超级沉闷的工作。Phalcon提供 [Phalcon\Tag](http://www.iphalcon.cn/api/Phalcon_Tag.html) 类来处理这些复杂而无趣的事情。



### 7.1 文档类型

Phalcon 提供 `Phalcon\Tag::setDoctype()` 方法可以设置输出内容的文档类型。此类型设置可能被其他的tag方法影响。

| 常量 | 对应的文档类型 |
| :-- | :-- |
| HTML32 | HTML 3.2 |
| HTML401_STRICT | HTML 4.01 严格模式 |
| HTML401_TRANSITIONAL | HTML 4.01 过渡模式 |
| HTML401_FRAMESET | HTML 4.01 Frameset |
| HTML5 | HTML 5 |
| XHTML10_STRICT | XHTML 1.0 严格模式 |
| XHTML10_TRANSITIONAL | XHTML 1.0 过渡模式 |
| XHTML10_FRAMESET | XHTML 1.0 Frameset |
| XHTML11 | XHTML 1.1 |
| XHTML20 | XHTML 2.0 |
| XHTML5 | XHTML 5 |

设置文档类型

```php
use Phalcon\Tag;

$this->tag->setDoctype(Tag::HTML401_STRICT);
```

### 7.2 生成链接

可使用如下的方法创建指向我们站内的超链接

#### 7.2.1 tag生成

```html
<!-- for the default route -->
<?= $this->tag->linkTo("products/search", "Search") ?>

<!-- with CSS attributes -->
<?= $this->tag->linkTo(["products/edit/10", "Edit", "class" => "edit-btn"]) ?>

<!-- for a named route -->
<?= $this->tag->linkTo([["for" => "show-product", "title" => 123, "name" => "carrots"], "Show"]) ?>
```

#### 7.2.2 Volt生成

```html
<!-- for the default route -->
{{ link_to("products/search", "Search") }}

<!-- for a named route -->
{{ link_to(["for": "show-product", "id": 123, "name": "carrots"], "Show") }}

<!-- for a named route with a HTML class -->
{{ link_to(["for": "show-product", "id": 123, "name": "carrots"], "Show", "class": "edit-btn") }}
```


### 7.3 创建表单

在Web应用中，表单是获取用户输入的重要工具，下面的例子显示了使用视图助手(tag)如何去生成一个简单的form表单。

#### 7.3.1 tag创建

```html
<!-- Sending the form by method POST -->
<?= $this->tag->form("products/search") ?>
    <label for="q">Search:</label>

    <?= $this->tag->textField("q") ?>

    <?= $this->tag->submitButton("Search") ?>
<?= $this->tag->endForm() ?>

<!-- Specifying another method or attributes for the FORM tag -->
<?= $this->tag->form(["products/search", "method" => "get"]); ?>
    <label for="q">Search:</label>

    <?= $this->tag->textField("q"); ?>

    <?= $this->tag->submitButton("Search"); ?>
<?= $this->tag->endForm() ?>
```

#### 7.3.2 Volt创建

```html
<!-- Specifying another method or attributes for the FORM tag -->
{{ form("products/search", "method": "get") }}
    <label for="q">Search:</label>

    {{ text_field("q") }}

    {{ submit_button("Search") }}
{{ endForm() }}
```

#### 7.3.3 生成html

```html
<form action="/store/products/search/" method="get">
    <label for="q">Search:</label>

    <input type="text" id="q" value="" name="q" />

    <input type="submit" value="Search" />
</form>
```

### 7.4 表单控件

Phalcon 提供了一系列的方法去生成例如文本域(text)，按钮(button)和其他的一些form表单元素。提供给所有方法(helper)的第一个参数都是需要创建的表单元素的名称(name属性)。当提交表单的时候，这个名称将被和form表单数据一起传输。在控制器中，你可以使用request对象 (`$this->request`) 的 `getPost()` 和 `getQuery()` 方法结合之前定义的名字(name属性)来获取到这些值。

#### 7.4.1 tag生成

```html
<?php echo $this->tag->textField("username") ?>

<?php echo $this->tag->textArea(
    [
        "comment",
        "This is the content of the text-area",
        "cols" => "6",
        "rows" => 20,
    ]
) ?>

<?php echo $this->tag->passwordField(
    [
        "password",
        "size" => 30,
    ]
) ?>

<?php echo $this->tag->hiddenField(
    [
        "parent_id",
        "value" => "5",
    ]
) ?>
```

#### 7.4.2 Volt生成

```php
{{ text_field("username") }}

{{ text_area("comment", "This is the content", "cols": "6", "rows": 20) }}

{{ password_field("password", "size": 30) }}

{{ hidden_field("parent_id", "value": "5") }}
```

### 7.5 select选择框

生成选择框(select)很简单,特别是当你已经把相关的数据存储在了PHP的关联数组中。生成select的方法是 `Phalcon\Tag::select()` 和 `Phalcon\Tag::selectStatic()` 。方法 `Phalcon\Tag::select()` 与 Phalcon\Mvc\Model 一起使用会更好。当然 `Phalcon\Tag::selectStatic() `也可以和PHP的数组一起工作。

#### 7.5.1 tag生成 

```php
<?php

$products = Products::find("type = 'vegetables'");

// Using data from a resultset
echo $this->tag->select(
    [
        "productId",
        $products,
        "using" => [
            "id",
            "name",
        ]
    ]
);

// Using data from an array
echo $this->tag->selectStatic(
    [
        "status",
        [
            "A" => "Active",
            "I" => "Inactive",
        ]
    ]
);
```
> 生成的html

```html
<select id="productId" name="productId">
    <option value="101">Tomato</option>
    <option value="102">Lettuce</option>
    <option value="103">Beans</option>
</select>

<select id="status" name="status">
    <option value="A">Active</option>
    <option value="I">Inactive</option>
</select>
```

#### 7.5.2 添加一个空选项(option)

在7.5.1的select方法中，添加`"useEmpty" => true,`到数组中，或定制如下

```php
// Creating a Select Tag with an empty option with default text
echo $this->tag->select(
    [
        "productId",
        $products,
        "using"      => [
            "id",
            "name",
        ],
        "useEmpty"   => true,
        "emptyText"  => "Please, choose one...", // 可选
        "emptyValue" => "@", // 可选
    ]
);
```

> 生成的html

```html
<select id="productId" name="productId">
    <option value="@">Please, choose one..</option>
    <option value="101">Tomato</option>
    <option value="102">Lettuce</option>
    <option value="103">Beans</option>
</select>
```

#### 7.5.3 volt 生成

```php
{# Creating a Select Tag with an empty option with default text #}
{{ select('productId', products, 'using': ['id', 'name'],
    'useEmpty': true, 'emptyText': 'Please, choose one...', 'emptyValue': '@') }}
```

### 7.6 input属性

#### 7.6.1 tag生成

```php
<?php $this->tag->textField(
    [
        "price",
        "size"        => 20,
        "maxlength"   => 30,
        "placeholder" => "Enter a price",
    ]
) ?>

```

#### 7.6.2 volt生成

```html
{{ text_field("price", "size": 20, "maxlength": 30, "placeholder": "Enter a price") }}
```

#### 7.6.3 html显示

```html
<input type="text" name="price" id="price" size="20" maxlength="30"
    placeholder="Enter a price" />
```

### 7.7 设置默认值

#### 7.7.1 通过控制器

使用MVC框架编程时的一个好习惯是给form元素在视图中设定一个明确的值。你可以直接使用 Phalcon\Tag::setDefault() 在控制器中设置这个值。这个方法为所有的视图助手的方法预先设定了一个值，如果任意一个视图助手方法有一个和此预设值相匹配的名字，这个值将会被使用，除非那个视图方法明确的指定了这个值。

```php
class ProductsController extends Controller
{
    public function indexAction()
    {
        $this->tag->setDefault("color", "Blue");
    }
}

echo $this->tag->selectStatic(
    [
        "color",
        [
            "Yellow" => "Yellow",
            "Blue"   => "Blue",
            "Red"    => "Red",
        ]
    ]
);
```

当这个选择框被生成的时候，”Blue”将被默认选中。

```html
<select id="color" name="color">
    <option value="Yellow">Yellow</option>
    <option value="Blue" selected="selected">Blue</option>
    <option value="Red">Red</option>
</select>
```

#### 7.7.2 直接设置值

所有的表单方法都支持参数”value”。你可以直接设置一个明确的值给表单方法。当这个值被明确设定的时候，任何通过 setDefault() 或者通过 请求(request) 所设置的值将被直接忽略。

### 7.8 标题

Phalcon\Tag 类提供了一些方法，让我们可以在控制器中动态地设置HTML文档的标题(title)。

```php
$this->tag->setTitle("Your Website");
$this->tag->prependTitle("Index of Posts - ");
```

```html
<head>
   <?php echo $this->tag->getTitle(); ?>
</head>
```

### 7.9 静态内容

Phalcon\Tag 也提供一些其他的方法去生成一些其他的标签，例如脚本(script),超链接(link)或者图片(img)。它可以帮助你很快的生成一些你应用中的静态资源

#### 7.9.1 图片

1. tag 方式

    ```php
    // Generate <img src="/your-app/img/hello.gif">
    echo $this->tag->image("img/hello.gif");
    
    // Generate <img alt="alternative text" src="/your-app/img/hello.gif">
    echo $this->tag->image(
        [
           "img/hello.gif",
           "alt" => "alternative text",
        ]
    );
```

2. volt 方式
    
    ```html
    {# Generate <img src="/your-app/img/hello.gif"> #}
    {{ image("img/hello.gif") }}
    
    {# Generate <img alt="alternative text" src="/your-app/img/hello.gif"> #}
    {{ image("img/hello.gif", "alt": "alternative text") }}
    ```


#### 7.9.2 样式表

1. tag 方式

    ```php
    // Generate <link rel="stylesheet" href="http://fonts.googleapis.com/css?family=Rosario" type="text/css">
    echo $this->tag->stylesheetLink("http://fonts.googleapis.com/css?family=Rosario", false);
    
    // Generate <link rel="stylesheet" href="/your-app/css/styles.css" type="text/css">
    echo $this->tag->stylesheetLink("css/styles.css");
    ```

2. Volt 方式

    ```html
    {# Generate <link rel="stylesheet" href="http://fonts.googleapis.com/css?family=Rosario" type="text/css"> #}
    {{ stylesheet_link("http://fonts.googleapis.com/css?family=Rosario", false) }}
    
    {# Generate <link rel="stylesheet" href="/your-app/css/styles.css" type="text/css"> #}
    {{ stylesheet_link("css/styles.css") }}
    ```

#### 7.9.3 js

1. tag 方式

    ```php
   // Generate <script src="http://localhost/javascript/jquery.min.js" type="text/javascript"></script>
    echo $this->tag->javascriptInclude("http://localhost/javascript/jquery.min.js", false);
    
    // Generate <script src="/your-app/javascript/jquery.min.js" type="text/javascript"></script>
    echo $this->tag->javascriptInclude("javascript/jquery.min.js");
    ```

2. volt 方式

    ```html
    {# Generate <script src="http://localhost/javascript/jquery.min.js" type="text/javascript"></script> #}
    {{ javascript_include("http://localhost/javascript/jquery.min.js", false) }}
    
    {# Generate <script src="/your-app/javascript/jquery.min.js" type="text/javascript"></script> #}
    {{ javascript_include("javascript/jquery.min.js") }}
    ```

## 八、Volt

[官方文档](http://www.iphalcon.cn/reference/volt.html)

## 九、Tools

## 十、附录




