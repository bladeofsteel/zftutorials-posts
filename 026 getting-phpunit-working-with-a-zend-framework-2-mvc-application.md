title: Начало работы PHPUnit с приложением Zend Framework 2
permalink: getting-phpunit-working-with-a-zend-framework-2-mvc-application
cite: "Эта статья демонстрирует как настроить PHPUnit для тестирования приложений на Zend Framework 2.
<blockquote>… пришло время попробовать поиграться с PHPUnit. Хотя я не думаю, что я одинок в желании использовать ZF2 и PHPUnit вместе, не так уж много людей говорят об этом.</blockquote>"
~~~~
__Источник:__ [Getting PHPUnit working with a Zend Framework 2 MVC application][1]  
__Автор:__ [Tom Oram][2]  
__Перевод:__ [Лобач Олег][3]  

[1]: http://devblog.x2k.co.uk/getting-phpunit-working-with-a-zend-framework-2-mvc-application/
[2]: http://devblog.x2k.co.uk/bio/
[3]: http://lobach.info/

Теперь, когда у меня есть небольшое _MVC-приложение_ на Zend Framework 2[^1], я решил, что пришло время попробовать поиграться с PHPUnit. Хотя я не думаю, что я одинок в желании использовать ZF2 и PHPUnit вместе, не так уж много людей говорят об этом.

[^1]: речь идет о приложении, созданном по руководству Роба Алена, найти которое можно в документации к Zend Framework 2, - **при. пер.**

Здесь я покажу как я запускал PHPUnit вместе с Zend Framework.

Первое, что я сделал, было создание каталога `tests` в корне моего проекта:

<pre class="lang:sh theme:neon font:consolas font-size:16 nums:false highlight:0 decode:true"><code>
composer.json
composer.phar
data
module
README.md
vendor
composer.lock
config
init_autoloader.php
public
tests
</code></pre>

Затем я создал конфигурационный файл `phpunit.xml` в каталоге `tests`, содержащий следующий XML:

<pre class="lang:xml"><code>
<phpunit
    bootstrap="./Bootstrap.php"
    colors="true"
    backupGlobals="false"
>
    <testsuites>
        <testsuite name="Test Suite">
            <directory>./</directory>
        </testsuite>
    </testsuites>
</phpunit>
</code></pre>

Ничего особенного в нем нет, основным является то, что я указал _phpunit_ использовать `Bootstrap.php` чтобы все настроить.

Так что следующим шагом мне нужно создать `Bootstrap.php`. 

Я пришел к выводу, что я мог бы использовать тот же файл `init_autoloader.php`, что используется `default/index.php` в _ZF2 Skeleton Application_ для настройки автозагрузчика.

Итак, я создал `Bootstrap.php`, содержащий следующее:

<pre class="lang:php"><code>
<?php

chdir(dirname(__DIR__));

include __DIR__ . '/../init_autoloader.php';
</code></pre>

Это выглядит как уловка, но теперь я могу загружать классы из `Zend Framework`, однако я все еще не могу загружать собственные классы из моего модуля _MVC-приложения_.

Я знал, что мне нужно как-нибудь вытащить конфигурацию MVC из приложения и после тщательного изучения кода я пришел к выводу, что мне необходимо добавить следующий код:

<pre class="lang:php"><code>
$configuration = include 'config/application.config.php';

$serviceManager = new ServiceManager(new ServiceManagerConfiguration($configuration));
$serviceManager->setService('ApplicationConfiguration', $configuration);
$moduleManager = $serviceManager->get('ModuleManager');
$moduleManager->loadModules();
</code></pre>

Он получает конфигурацию приложения из файла `config/application.config.php` и затем загружает конфигруацию для модулей, перечисленных там. Так что теперь целиком мой `Bootstrap.php` выглядит так:

<pre class="lang:php"><code>
<?php
use Zend\ServiceManager\ServiceManager,
    Zend\Mvc\Service\ServiceManagerConfiguration;

chdir(dirname(__DIR__));

include __DIR__ . '/../init_autoloader.php';

$configuration = include 'config/application.config.php';

$serviceManager = new ServiceManager(new ServiceManagerConfiguration($configuration));
$serviceManager->setService('ApplicationConfiguration', $configuration);
$moduleManager = $serviceManager->get('ModuleManager');
$moduleManager->loadModules();
</code></pre>

Теперь мои классы из моих модулей могут быть легко загружены, великолепно!

[stextbox id="info" caption="Обновление"]После релиза я понял, что эта настройка перенесена в метод **init ()** **Zend\Mvc\Application**, поэтому `Bootstrap.php` может быть уменьшен до:

<pre class="lang:php"><code>
<?php
chdir(dirname(__DIR__));

include __DIR__ . '/../init_autoloader.php';

Zend\Mvc\Application::init(include 'config/application.config.php');
</code></pre>
Что намного аккуратнее и также вызывает метод **bootstrap()** приложения, что наверняка пригодится позже.[/stextbox]

Следующим шагом было написание теста. Я решил, что структура каталогов в моём каталоге `tests` будет посторять древо основного кода, поэтому я создал `module/Album/src/Album/Model/` и поместил туда новый файл, назвав его `AlbumTest.php`.

Я написал следующий код теста:

<pre class="lang:php"><code>
<?php

use Album\Model\Album,
    Zend\InputFilter\InputFilterInterface;

class AlbumTest extends \PHPUnit_Framework_TestCase
{
    protected $a;

    public function setUp()
    {
        $this->a = new Album;
    }

    /**
     * @expectedException Album\Model\AlbumException
     * @expectedExceptionMessage Not used
     */
    public function testSetInputFilter()
    {
        $if = $this->getMock('Zend\InputFilter\InputFilterInterface');
        $this->a->setInputFilter($if);
    }
 
    public function testGetInputFilter()
    {
        $if = $this->a->getInputFilter();
 
        $this->assertInstanceOf("Zend\InputFilter\InputFilter", $if);
        return $if;
    }
 
    /**
     * @depends testGetInputFilter
     */
    public function testInputFilterValid($if)
    {
        $this->assertEquals(3, $if->count());
 
        $this->assertTrue($if->has('title'));
        $this->assertTrue($if->has('artist'));
        $this->assertTrue($if->has('id'));
    }
}
</code></pre>

Я не хочу углубляться в детали того, что этот тест делает, все это можно найти в документации на PHPUnit. Одна вещь, которую я поменял: `Album::setInputFilter` бросает исключение `AlbumException` вместо `Exception`. Причиной для этого послужило то, что _PHPUnit 3.6_ не нравится проверка классов `Exception`, вероятно это будет исправлено в _3.7_.

Так или иначе, когда я перехожу в мой каталог тестов и набираю `phpunit` я получаю следующий вывод:

<pre class="lang:sh theme:neon font:consolas font-size:16 nums:false highlight:0 decode:true"><code>
PHPUnit 3.6.11 by Sebastian Bergmann.
Configuration read from /home/tom/workspace/ZF2Tutorial/tests/phpunit.xml
...
Time: 0 seconds, Memory: 5.25Mb
OK (3 tests, 7 assertions)
</code></pre>

Это успех!

В следующий раз настанет время разобраться как использовать PHPUnit с классами для доступа к БД и классами контроллеров.