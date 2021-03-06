---
title: 自定义php模板引擎
date: 2016-08-03 16:48:30
categories: 
- lnmp
tags: 
- php
- mvc
---

模板引擎是一个web框架必不可少的组件，本文主要是实现了一个简单的php模板引擎

# MVC和模板引擎
模板引擎的思想是来源于MVC（Model View Controller）模型，即模型层、视图层、控制器层。  
在web端，模型层为数据库的操作；视图层就是模板引擎；Controller就是PHP对数据和请求的各种操作。模板引擎就是为了将视图层和其他层分离开来，使php代码和html代码不会混杂在一起。否则随着代码量的增多，web应用代码的可读性和可维护性将会变的越来越差。

***

# 模板引擎原理
大部分的模板引擎原理都差不多，核心就是利用正则表达式解析模板，将约定好的特定的标识语句编译成php语句，然后调用时只需要include编译后的文件，这样就讲php语句和html语句分离开来了。  
如果数据基本没有变动，例如博客，甚至可以更进一步将php的输出输出到缓冲区，然后将模板编译成静态的html文件，这样请求时，就是直接打开静态的html文件，请求速度大大加快。  
一个简单的自定义模板引擎就是两个类，**模板类**和**编译类**。 

***

# 实现
## 编译类

这个简单的编译类主要有包括:  

* 预定义语法
* 处理include的方法  
* 解析预定于语法的方法  
* 解析js的方法  

代码如下:

```php
class CompileClass
{
    private $template;      // 待编译文件
    private $content;       // 需要替换的文本
    private $compile_file;       // 编译后的文件
    private $left         = '{';       // 左定界符
    private $right        = '}';      // 右定界符
    private $include_file = [];        // 引入的文件
    private $config;        // 模板的配置文件
    private $subjects     = [];     // 需要替换的表达式
    private $replaces     = [];     // 替换后的字符串
    private $vars         = [];

    public function __construct($template, $compile_file, $config) {}

    public function compile()
    {
        $this->compileInclude();
        $this->compileVar();
        $this->compileStaticFile();
        file_put_contents($this->compile_file, $this->content);
    }

    // 处理include
    public function compileInclude() {}

    // 处理各种赋值和基本语句
    public function compileVar() {}

    // 对静态的JavaScript进行解析    
    public function compileStaticFile() {}
}
```

### 预定义语法
在这个自定义模板引擎中，左右定界符就是大括号，具体的解析规则如下

```php
private $left         = '{';    // 左定界符
private $right        = '}';    // 右定界符
private $subjects     = [];     // 需要替换的表达式
private $replaces     = [];     // 替换后的字符串
    
// 需要替换的正则表达式
$this->subjects[] = "/$this->left\s*\\$([a-zA-Z_\x7f-\xff][a-zA-Z0-9_\xf7-\xff]*)\s*$this->right/";
$this->subjects[] = "/$this->left\s*(loop|foreach)\s*\\$([a-zA-Z_\x7f-\xff][a-zA-Z0-9_\xf7-\xff]*)\s*$this->right/";
$this->subjects[] = "/$this->left\s*(loop|foreach)\s*\\$([a-zA-Z_\x7f-\xff][a-zA-Z0-9_\xf7-\xff]*)\s+"
    . "as\s+\\$([a-zA-Z_\x7f-\xff][a-zA-Z0-9_\xf7-\xff]*)$this->right/";
$this->subjects[] = "/$this->left\s*\/(loop|foreach|if)\s*$this->right/";
$this->subjects[] = "/$this->left\s*if(.*?)\s*$this->right/";
$this->subjects[] = "/$this->left\s*(else if|elseif)(.*?)\s*$this->right/";
$this->subjects[] = "/$this->left\s*else\s*$this->right/";
$this->subjects[] = "/$this->left\s*([a-zA-Z_\x7f-\xff][a-zA-Z0-9_\xf7-\xff]*)\s*$this->right/";

// 替换后的字符串         
$this->replaces[] = "<?php echo \$\\1; ?>";
$this->replaces[] = "<?php foreach((array)\$\\2 as \$K=>\$V) { ?>";
$this->replaces[] = "<?php foreach((array)\$\\2 as &\$\\3) { ?>";
$this->replaces[] = "<?php } ?>";
$this->replaces[] = "<?php if(\\1) { ?>";
$this->replaces[] = "<?php } elseif(\\2) { ?>";
$this->replaces[] = "<?php } else { ?>";
$this->replaces[] = "<?php echo \$\\1; ?>";
```

上面的解析规则包含了基本的输出和一些常用的语法，if、foreach等。利用preg_replace函数就能对模板文件进行替换。具体使用和解析情况如下

### 模板语法
```php
{$data}
{foreach $vars}
    {if $V == 1 }
        <input value="{V}">
    {elseif $V == 2}
        <input value="123123">
    {else }
        <input value="sdfsas是aa">
    {/if}
{/foreach}

{ loop $vars as $var}
    <input value="{var}">
{ /loop }
```

### 解析后
```php
<?php echo $data; ?>
<?php foreach((array)$vars as $K=>$V) { ?>
    <?php if( $V == 1) { ?>
        <input value="<?php echo $V; ?>">
    <?php } elseif( $V == 2) { ?>
        <input value="123123">
    <?php } else { ?>
        <input value="sdfsas是aa">
    <?php } ?>
<?php } ?>

<?php foreach((array)$vars as &$var) { ?>
    <input value="<?php echo $var; ?>">
<?php } ?>
```

编译类的原理就是正则替换，将一些预定义语法替换成php的语法。这里只举了预定义语法的解析的例子，include和js解析亦是如是。

## 模板类
一个简单的模板类的主要方法如下，包括:

* 变量注入，assign和assignArray
* 设置模板配置
* 编译、展示模板
* 数据缓存

```php
class Template
{
    // 配置数组    
    private        $_array_config = [
        'root'         => '',               // 文件根目录
        'suffix'       => '.html',          // 模板文件后缀
        'template_dir' => 'templates',      // 模板所在文件夹
        'compile_dir'  => 'templates_c',    // 编译后存放的文件夹
        'cache_dir'    => 'cache',          // 静态html存放地址
        'cache_htm'    => false,            // 是否编译为静态html文件
        'suffix_cache' => '.htm',           // 设置编译文件的后缀
        'cache_time'   => 7200,             // 自动更新间隔
        'php_turn'     => true,             // 是否支持原生php代码
        'debug'        => 'false',
    ];
    private        $_value        = [];
    private        $_compileTool;      // 编译器
    public         $file;        // 模板文件名
    public         $debug         = [];        // 调试信息

    public function __construct($array_config = []) {}

    // 单步设置配置文件
    public function setConfig($key, $value = null){}

    // 注入单个变量
    public function assign($key, $value) {}

    // 注入数组变量
    public function assignArray($array) {}

    // 是否开启缓存
    public function needCache() {}

    // 如果需要重新编译文件
    public function reCache() {}

    // 显示模板
    public function show($file) {}
}
```

整个模板类的工作流程就是先实例化模板类对象，然后利用assign和assignArray方法给模板中的变量赋值，通过调用show方法，将模板和配置文件传入编译类的实例化对象中，最终直接include编译后的php、html混编文件，显示输出。详细的代码如下

```php
public function show($file)
    {
        $this->file = $file;
        if (!is_file($this->path())) {
            exit("找不到对应的模板文件");
        }

        $compile_file = $this->_array_config['compile_dir'] . md5($file) . '.php';
        $cache_file   = $this->_array_config['cache_dir'] . md5($file) . $this->_array_config['suffix_cache'];

        // 如果需要重新编译文件
        if ($this->reCache() === false) {
            $this->debug['cached'] = 'false';
            $this->_compileTool    = new CompileClass($this->path(), $compile_file, $this->_array_config);

            if ($this->needCache() === true) {
                // 输出到缓冲区
                ob_start();
            }
            // 将赋值的变量导入当前符号表
            extract($this->_value, EXTR_OVERWRITE);

            if (!is_file($compile_file) or filemtime($compile_file) < filemtime($this->path())) {
                $this->_compileTool->vars = $this->_value;
                $this->_compileTool->compile();
                include($compile_file);
            } else {
                include($compile_file);
            }

            // 如果需要编译成静态文件
            if ($this->needCache() === true) {
                $message = ob_get_contents();
                file_put_contents($cache_file, $message);
            }
        } else {
            readfile($cache_file);
            $this->debug['cached'] = 'true';
        }
        $this->debug['spend'] = microtime(true) - $this->debug['begin'];
        $this->debug['count'] = count($this->_value);
        $this->debugInfo();
    }
```

在show方法中，我首先判断模板文件存在，然后利用MD5编码生成编译文件和缓存文件的文件名。然后就是判断是否需要进行编译，判断的依据是看编译文件是否存在和编译文件的写入时间是否小于模板文件。如果需要编译，就利用编译类进行编译，生成一个php文件。然后只需要include这个编译文件就好了。  
为了加快模板的载入，可以将编译后的文件输出到缓冲区中，也就是ob_start()这个函数，所有的输出将不会输出到浏览器，而是输出到默认的缓冲区，在利用ob_get_contents()将输出读取出来，保存成静态的html文件。  

***
# 使用示例
测试代码如下

```php
require('Template.php');

date_default_timezone_set('PRC');

$config = array(
    'debug' => true,
    'cache_htm' => true,
);

$tpl = new Template($config);
$tpl->assign('data', microtime(true));
$tpl->assign('vars', array(1,2,3));
$tpl->assign('title', "hhhh");
$tpl->show('test');
```

其中test.html模板代码如下 

```html
{include file="header.html"}
    <body>
{$data}
{foreach $vars}
    {if $V == 1 }
        <input value="{V}">
    {elseif $V == 2}
        <input value="123123">
    {else }
        <input value="sdfsas是aa">
    {/if}
{/foreach}

{ loop $vars as $var}
    <input value="{var}">
{ /loop }
{!123!}
    </body>
{include file="footer.html"}
```

最终输出

```php
// 编译文件
<!DOCTYPE html>
<html>
    <head>
        <title><?php echo $title; ?></title>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
    </head>
    <body>
<?php echo $data; ?>
<?php foreach((array)$vars as $K=>$V) { ?>
    <?php if( $V == 1) { ?>
        <input value="<?php echo $V; ?>">
    <?php } elseif( $V == 2) { ?>
        <input value="123123">
    <?php } else { ?>
        <input value="sdfsas是aa">
    <?php } ?>
<?php } ?>

<?php foreach((array)$vars as &$var) { ?>
    <input value="<?php echo $var; ?>">
<?php } ?>
<script src="123?t=1510672390"></script>
    </body>
</html>

// 包含具体数据的缓存文件
<!DOCTYPE html>
<html>
    <head>
        <title>hhhh</title>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
    </head>
    <body>
        1510672390.5514            
        <input value="1">
        <input value="123123">
        <input value="sdfsas是aa">
        <input value="1">
        <input value="2">
        <input value="3">
        <script src="123?t=1510672390"></script>
    </body>
</html>
```

完整代码可见:[github](https://github.com/CzRzChao/SimplePHPTemplate)