__Источник:__ [PHPUnit: Testing Zend Framework Controllers][1]
__Автор:__ [Federico Cargnelutti][2]
__Перевод:__ [Лобач Олег](http://lobach.info/)

 [1]: http://phpimpact.wordpress.com/2008/12/27/phpunit-testing-zend-framework-controllers/
 [2]: http://phpimpact.wordpress.com/

Тестирование Веб-приложений — это комплексная задача, потому что веб-приложение создается из нескольких логических слоев. Модульное тестирование контроллера Zend Framework может быть весьма трудной задачей, особенно для тех, кто слабо знаком с Zend Framework.

Вы можете тестировать свои контроллеры действий используя [Zend_Test][3] и/или [PHPUnit][4]. Zend_Test позволяет вам имитировать запросы, передавать тестовые данные, контролировать вывод вашего приложения и в целом убедиться в том, что ваш код делает именно то, что должен делать. Вам решать, какой из них использовать. Если вы не можете выбрать один из них, то можете использовать оба. Если вы только знакомитесь с тестированием с помощью Zend_Test, то [эта статья][5] будет лучшим местом старта.

 [3]: http://framework.zend.com/manual/en/zend.test.html
 [4]: http://www.phpunit.de/
 [5]: http://weierophinney.net/matthew/archives/190-Setting-up-your-Zend_Test-test-suites.html

Фреймворк PHPUnit может показаться очень знакомым тем разработчикам, которые пришли из Java. Разработчики PHPUnit черпали вдохновение из JUnit - тестовом фреймворке для платформы Java, поэтому вы будете чувствовать себя как дома при использовании PHPUnit если вам уже приходилось сталкиваться с JUnit или одним из его клонов.

Конечно, никто не запрещает вам использовать системы бок о бок (даже в одном и том же приложении). В конце концов, большинство проектов так и будет использовать.

## Использование PHPUnit

Во-первых, вам необходимо создать структуру каталогов:

<pre lang="default" class="font:consolas font-size:16 highlight:0 decode:true"><code>app/
    config/
    controllers/
        ExampleController.php
    models/
    views/
lib/
    Zend/
public/
tests/
    controllers/
        AllTests.php
        ExampleControllerTest.php
    lib/
    AllTests.php
    bootstrap.php</code></pre>

Тестовый набор нуждается в некоторой информации об окружении, и обычно эта информация находится в файле bootstrap.php. Самым большим отличием этого файла от одного из используемых в вашем приложении является то, что Фронт-контроллер не выполняет диспетчеризацию объекта запроса:

**tests/bootstrap.php** [ <a href="http://phpimpact.codepad.org/3XY6HY1b" target="_blank">Открыть в Codepad</a> ]

<pre><code class="php"><?php
/* Start output buffering */
ob_start();

/* Report all errors directly to the screen for simple diagnostics in the dev environment */
error_reporting( E_ALL | E_STRICT );
ini_set('display_startup_errors', 1);
ini_set('display_errors', 1);
date_default_timezone_set('Europe/London');

/* Determine the root and library directories of the application */
$appRoot = dirname(__FILE__) . '/..';
$libDir = "$appRoot/lib";
$path = array($libDir, get_include_path());
set_include_path(implode(PATH_SEPARATOR, $path));

define('APPLICATION_PATH', $appRoot . '/app');
define('APPLICATION_ENVIRONMENT', 'dev');

require_once "Zend/Loader.php";
Zend_Loader::registerAutoload();

$front = Zend_Controller_Front::getInstance();
$front->throwExceptions(true);
$front->setParam('noViewRenderer', true);
$front->setParam('env', APPLICATION_ENVIRONMENT);
$front->setRequest(new Zend_Controller_Request_Http());
$front->returnResponse(true);

$router = $front->getRouter();
include APPLICATION_PATH . '/config/routes.php';
$router->addRoutes($routes);
$router->setParams($front->getParams());

$dispatcher = $front->getDispatcher();
$dispatcher->setParams($front->getParams());
$dispatcher->setResponse($front->getResponse());

$router->route($front->getRequest());</code></pre>

Обратите внимание! Отключение помощника `ViewRenderer` является не обязательным. Однако, вам должно быть известно, что использование класса `Zend_Controller_Action_Helper_ViewRenderer` может привести к снижению производительности. Подробнее об этом можно прочесть [здесь][6].

 [6]: http://phpimpact.wordpress.com/2008/09/16/zend-framework-controller-22-drop-in-responsiveness/

Класс `PHPUnit_Framework_TestSuite` фреймворка PHPUnit позволяет вам организовать тесты в иерархические наборы тестов:

**tests/AllTests.php** [ <a href="http://phpimpact.codepad.org/8EQspGqq" target="_blank">Открыть в Codepad</a> ]

<pre><code class="php"><?php
require_once dirname(__FILE__) . '/bootstrap.php';
require_once dirname(__FILE__) . '/controllers/AllTests.php';

class AllTests
{
    public static function main()
    {
        $parameters = array();
        PHPUnit_TextUI_TestRunner::run(self::suite(), $parameters);
    }

    public static function suite()
    {
        $suite = new PHPUnit_Framework_TestSuite('My Application');
        $suite->addTest(ControllersAllTests::suite());
        return $suite;
    }
}

AllTests::main();</code></pre>

**tests/controllers/AllTests.php** [ <a href="http://phpimpact.codepad.org/iDFGH0nf" target="_blank">Открыть в Codepad</a> ]

<pre><code class="php"><?php
require_once dirname(__FILE__) . '/ExampleControllerTest.php';

class ControllersAllTests
{
    public static function main()
    {
        PHPUnit_TextUI_TestRunner::run(self::suite());
    }

    public static function suite()
    {
        $suite = new PHPUnit_Framework_TestSuite('My Application - Controllers');
        $suite->addTestSuite('ExampleControllerTestCase');
        return $suite;
    }
}</code></pre>

## Написание модульных тестов

Из-за довольно [странных причин][7] эта часть не описана в документации. Вот что вам нужно сделать до написания теста:

 [7]: http://sebastian-bergmann.de/archives/779-PHP-Has-No-Culture-of-Testing.html

1.  Подключить контроллер, который вы собираетесь тестировать.
2.  Расширить контроллер действий (унаследовавшись от него).
3.  Сбросить состояние экземпляра фронт-контроллера.
4.  Указать путь к тестируемому контроллеру действий.
5.  Установить объекты Запроса и Ответа.
6.  Создать экземпляр тестируемого объекта.

Пример:

**tests/controllers/ExampleControllerTest.php** [ <a href="http://phpimpact.codepad.org/LgS7T5ly" target="_blank">Открыть в Codepad</a> ]

<pre><code class="php"><?php
require_once APPLICATION_PATH . '/controllers/ExampleController.php';

class ExampleControllerTest extends ExampleController
{
    public function __construct($url = null)
    {
        $front = Zend_Controller_Front::getInstance();
        $front->resetInstance();
        $front->setControllerDirectory(APPLICATION_PATH . '/controllers');
        $front->setRequest(new Zend_Controller_Request_Http($url));
        $front->setResponse(new Zend_Controller_Response_Http());

        parent::__construct($front->getRequest(), $front->getResponse());
    }
}</code></pre>

Вся магия происходит внутри класса `ExampleControllerTest`. Он делает так, что контроллер действий думает, что был вызван фронт-контроллером в цикле диспетчеризации. Единственный путь сделать это — создание экземпляра контроллера действий без диспетчеризации запроса. Получение экземпляра контроллера действий дает вам больше контроля и гибкости, особенно при тестировании веб-сервисов.

А теперь пришло время создать наш первый тестовый набор. Тестовый набор это класс, наследуемый от `PHPUnit_Framework_TestCase`, содержащий тестовые методы, определяемые по префиксу “test” в названии метода.

[ <a href="http://phpimpact.codepad.org/4Ldq4ORj" target="_blank">Открыть в Codepad</a> ]

<pre><code class="php">require_once APPLICATION_PATH . '/controllers/ExampleController.php';

class ExampleControllerTest extends ExampleController
{
    ...
}

class ExampleControllerTestCase extends PHPUnit_Framework_TestCase
{
    public function testDefaultAction()
    {
        $controller = new ExampleControllerTest();
        $isDispatched = $controller->indexAction();

        $this->assertTrue($isDispatched);
    }

    public function testFirstAction()
    {
        $url = 'http://localhost/example/first';
        $controller = new ExampleControllerTest($url);
        $controller->firstAction();
        $errorMsg = $controller->getRequest()->getParam('error_message', null);

        $this->assertEquals(null, $errorMsg);
    }

    public function testGetParameterName()
    {
        $url = 'http://localhost/example/first/fed';
        $controller = new ExampleControllerTest($url);
        $name = $controller->getRequest()->getParam('name', null);

        $this->assertEquals('fed', $name);
    }

    public function testGetNameMethod()
    {
        $url = 'http://localhost/example/first/fed';
        $controller = new ExampleControllerTest($url);
        $name = $controller->getName();

        $this->assertEquals('fed', $name);
    }
}</code></pre>

## Запуск тестов

<pre lang="default" class="font:consolas font-size:16 highlight:0 decode:true"><code>federico@tests$ phpunit AllTests
PHPUnit 3.3.8 by Sebastian Bergmann.
.....
Time: 0 seconds
OK (4 tests, 4 assertions)</code></pre>

Если тестирование завершится неудачно, то вы увидите подробную информацию о проваленном тесте. По желанию, вы можете <a href="http://hudson.gotdns.com/wiki/display/HUDSON/Phing+Plugin" target="_blank">подключить Phing в Hudson</a> и автоматизировать выполнение этой задачи. Если есть вопросы — обращайтесь.