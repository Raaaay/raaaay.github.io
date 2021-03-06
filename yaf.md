## Yaf简明教程
### 安装&配置
1. 从[github地址](https://github.com/laruence/yaf/releases)中选择最新的稳定release，拷贝到php source文件夹下。
2. 编译yaf.so扩展包。编译命令示例：
```shell
./PHP_PATH/bin/phpize
./configure --with-php-config=PHP_PATH/bin/php-config
make && make install
```
编译完成后将yaf.so放至扩展目录下。

3. 配置php.ini文件，添加yaf扩展。示例配置如下：
```ini
extension=yaf.so
# set the runtime environ to product
# more settings see http://php.net/manual/en/yaf.configuration.php
yaf.environ=product
```

4. 重启php-fpm。
### hello world
1. 在自己的workspace中新建一个工程文件夹，本示例中为yaf-test。
2. 在yaf-test中新建一些必要的工程文件和文件夹。本节示例代码都可以在[http://php.net/manual/en/yaf.tutorials.php](http://php.net/manual/en/yaf.tutorials.php) 中找到。
一个典型的yaf工程目录结构如下：
```text
- index.php 
- .htaccess 
+ conf
  |- application.ini //application config
- application/
  - Bootstrap.php   
  + controllers
     - Index.php //default controller
  + views    
     |+ index   
        - index.phtml //view template for default action
  + modules 
  - library
  - models  
  - plugins
```
3. 要实现简单的hello world，只需要几行代码即可。
```php
# index.php
define("APPLICATION_PATH",  dirname(__FILE__));
$app  = new Yaf_Application(APPLICATION_PATH . "/conf/application.ini");
$app->bootstrap()->run();
```
```php
# application/Bootstrap.php
class Bootstrap extends Yaf_Bootstrap_Abstract {
  public function _initConfig(Dispatcher $dispatcher){}
  public function _initPlugin(Dispatcher $dispatcher){}
}
```
```php
# application/controllers/index.php
class IndexController extends Yaf_Controller_Abstract {
  public function indexAction() {
    echo "hello world!";
  }
}
```
4. 配置conf/application.ini配置文件。在运行时，会自动选择与yaf.environ名称相同的配置节点。可以在线上线下设置不同的yaf.environ实现线上线下差异化配置。
```ini
[common]
application.directory=yaf-test
[product:common]
# product节点配置继承common
```
5. 配置自己的web server。此处给出nginx的配置示例。
```nginx
location / {
  root yaf-test;
  rewrite ^(.*)$ /index.php break;
  fastcgi_pass 127.0.0.1:9000;
  fastcgi_param SCRIPT_FILENAME yaf-test/index.php;
  include fastcgi_params;
}
```
配置完成后，启动nginx，即可在浏览器中看到效果。
