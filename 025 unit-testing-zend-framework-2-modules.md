Тестирование модулей в Zend Framework 2
-----------------------------------------
Как тестировать модули в ZF2? Ответ на этот вопрос вы найдете в переводе поста Robert Basic.
<blockquote>Переписывая этот блог на Zend Framework 2, я решил написать несколько модульных тестов. Не то, что бы текущая база кода не имела модульных тестов, просто их недостаточно много... В любом случае, я хотел бы показать как получить модульные тесты для модулей и обеспечить их работу, а так же о том, как добавить Mockery, и как mock-объекты могут оказать значительную помощь.</blockquote>
-----------------------------------------
__Источник:__ [Unit testing Zend Framework 2 modules ][1]  
__Автор:__ [Robert Basic][2]  
__Перевод:__ [Лобач Олег][3]  

[1]: http://robertbasic.com/blog/unit-testing-zend-framework-2-modules
[2]: http://robertbasic.com/
[3]: http://lobach.info/

Переписывая этот блог на Zend Framework 2, я решил написать несколько модульных тестов. Не то, что бы текущая база кода не имела модульных тестов, просто их недостаточно много... В любом случае, я хотел бы показать как получить модульные тесты для модулей и обеспечить их работу, а так же о том, как добавить [Mockery][4], и как mock-объекты могут оказать значительную помощь. Некоторые части, показанные здесь, можно написать чище/лучше, особенно кусочки автозагрузки, но пока меня устраивает текущее состояние.

[4]: https://github.com/padraic/mockery

Файл `phpunit.xml` достаточно прост:

<pre class="lang:xml"><code>
<phpunit bootstrap='./bootstrap.php' colors='true'>
    <testsuite name='ZF2 Module Test Suite'>
        <directory>.</directory>
    </testsuite>
    <filter>
        <whitelist>
            <directory suffix='.php'>../src/</directory>
        </whitelist>
    </filter>
    <listeners>
        <listener class="\Mockery\Adapter\Phpunit\TestListener"
            file="Mockery/Adapter/Phpunit/TestListener.php"></listener>
    </listeners>
</phpunit>
</code></pre>

`TestListener` [Mockery][4], как [я успел убедиться на собственной шкуре][5], нужен для корректной работы Mockery. Вы можете добавить дополнительные вещи, например, генерацию отчетов о покрытии кода и тому подобное.

[5]: https://github.com/padraic/mockery/issues/83

В файле `bootstrap.php` мы настраиваем автозагрузку для модулей, библиотеки ZF2 и Mockery:

<pre class="lang:php"><code>
putenv('ZF2_PATH=' . __DIR__ . '/../../../vendor/ZF2/library');
include_once __DIR__ . '/../../../init_autoloader.php';
set_include_path(implode(PATH_SEPARATOR, array(
    '.',
    __DIR__ . '/../src',
    __DIR__ . '/../../SomeRequiredModule/src',
    __DIR__ . '/../../../vendor',
    get_include_path(),
)));
spl_autoload_register(function($class) {
    $file = str_replace(array('\\', '_'), DIRECTORY_SEPARATOR, $class) . '.php';
    if (false === ($realpath = stream_resolve_include_path($file))) {
        return false;
    }
    include_once $realpath;
});
$loader = new \Mockery\Loader;
$loader->register();
</code></pre>

Он предполагает, что текущий тестируемый модуль живет внутри приложения ZF2. Если это не так, вам возможно потребуется исправить пути соответствующим образом. Он так же предполагает, что Mockery располагается в каталоге `vendor/`.

## Тестирование сервисного слоя

Не хочу лезть в драку по поводу терминологии, но для меня сервисный слой - это слой, работающий между слоем контроллера и слоем базы данных. Это позволяет сохранять остальные слои свободными от бизнес-логики и упрощает тестирование. Эти сервисы реализуют `Zend\ServiceManager\ServiceLocatorAwareInterface`, что значительно упрощает модульное тестирование, так как легко заменить конкретный объект моком.

Давайте предположим, что у нас есть сервис "статья", который мы можем использовать для получения последних статей. Сервис статей сам не взаимодействует с базой данных, но вызывает `AbstractTableGateway`, который выполняет все работы с базой данных. Тестовый случай (test case) для этого сервиса статей, чтобы избежать обращений к базе данных, должен "подделывать" `AbstractTableGateway` и использовать ServiceManager для замены конкретной реализации mock-объектом. Пример тестового случая для этого сервиса статей может выглядеть как-то так:

<pre class="lang:php"><code>
namespace BlogModule\Service;

use PHPUnit_Framework_TestCase as TestCase;
use Zend\ServiceManager\ServiceManager;
use Zend\Db\ResultSet\ResultSet;
use \Mockery as m;

class PostTest extends TestCase
{
    protected $postService;
    
    /**
    * @var Zend\ServiceManager\ServiceLocatorInterface
    */
    protected $serviceManager;

    public function setup()
    {
        $this->postService = new Post;
        $this->serviceManager = new ServiceManager;
        $this->postService->setServiceLocator($this->serviceManager);
    }

    public function testGetRecentPosts()
    {
        $mock = m::mock('Blog\Model\Table\Post');
        $this->serviceManager->setService('blogModelTablePost', $mock);
        $result = array(
            array(
                'id' => 1,
                'title' => 'Foo',
            ),
        );
        $resultSet = new ResultSet;
        $resultSet->initialize($result);
        $mock->shouldReceive('getRecentPosts')
            ->once()
            ->andReturn($resultSet);
        $posts = $this->postService->getRecentPosts();
        $this->assertSame($posts, $resultSet);
    }
}
</code></pre>

На строке 21 мы устанавливаем сервис-менеджер, который будет использоваться в сервисе статей, на строке 26 мы создаем mock-объект, и на строке 27 мы устанавливаем этот mock-объект в сервис-менеджер. Мы указываем mock-объекту некоторые ожидания - какой метод должен быть вызван, сколько раз и что он должен вернуть. Наконец, мы вызываем метод сервиса статей, который хотим протестировать, и проверям что возвращаемый результат верен.

## Тестирование слоя базы данных

Для тестирования слоя базы данных, который реализует `AbstractTableGateway`, я использовал небольшой... трюк. Я на самом деле не проверяю что именно возвращается из базы данных, а проверяю что вызывается правильный объект `Sql\Select` с правильными параметрами в правильном порядке. Это, в свою очередь, означает, что я доверяю базовому коду `Zend\Db` ктоторый, в конечном итоге, соберет правильный SQL-запрос, но мне так же не приходится возиться с настройкой тестовой базы данных, к тому же тесты выполняются быстре, так как они не выполняют реальных запросов к базе данных. Пример тестового случая, продолжая наш пример получения последних статей:

<pre class="lang:php"><code>
namespace Blog\Model\Table;

use PHPUnit_Framework_TestCase as TestCase;
use Zend\ServiceManager\ServiceManager;
use Zend\Db\Sql\Select;
use \Mockery as m;

class PostTest extends TestCase
{
    protected $postTable;
    protected $select;
    protected $tableName = 'blog_posts';
    public function setup()
    {
        $adapter = $this->getAdapterMock();
        $this->postTable = new Post($adapter);
        $this->select = m::mock(new Select($this->tableName));
        $this->postTable->setSelect($this->select);
    }

    public function testGetRecentPosts()
    {
        $this->select->shouldReceive('from')
            ->once()
            ->with($this->tableName)
            ->andReturn($this->select);
        $this->select->shouldReceive('where')
            ->once()
            ->with(array('published = ?' => 1))
            ->andReturn($this->select);
        $this->select->shouldReceive('order')
            ->once()
            ->with('id DESC')
            ->andReturn($this->select);
        $this->select->shouldReceive('limit')
            ->once()
            ->with(10)
            ->andReturn($this->select);
        $this->postTable->getRecentPosts();
    }
}
</code></pre>

Здесь мы создаем _mock_ адаптера (метод `getAdapterMock` можно посмотреть в [этом гисте][6]) и используем его в нашей реализации `AbstractTableGateway`. Мы так же создаем мок объекта `Sql\Select` и мы устанавливаем ожидания этому моку объекта Select. Как я уже говорил, это может быть не самый лучший способ тестирования, но он помогает мне отловить ошибку когда я забываю добавить условие _where_ в конкретном объекте Select. Ура Mockery! Ох, и, пожалуйста, обратите внимание, что мок адаптера может работать не во всех случаях, но опять же, для меня до сих пор это работает хорошо.

[6]: https://gist.github.com/3717485

Еще одну вещь, вероятно, было бы интересно показать как тестировать, и это _контроллеры_, но я не удосужился написать какие-либо контроллеры, так что я, наверное, оставлю это часть номер 2. Или сделайте это сами, в качестве домашнего задания.

Счастливого хака!