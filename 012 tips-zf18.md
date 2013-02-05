В комментариях к учебнику [в блоге Роба Алена](http://akrabat.com/ "Блог Роба Алена") (кстати, он его уже обновил до последней текущей версии ZF - 1.8) [задают вопрос](http://akrabat.com/zend-framework-tutorial/#comment-25889 "Комментарий к учебнику с вопросом о доступе к конфигурационным параметрам"):

<blockquote markdown="1">Как в новой версии фреймворка применять конфигурационный файл?
Раньше вопрошающий устанавливал переменную в своем конфигурационном файле:  
`cms.max.feeds = 10`  

А в загрузочном файле писал следующее:

<pre><code class="php">
$configuration = new Zend_Config_Ini(
    APPLICATION_PATH . '/config/app.ini',
    APPLICATION_ENVIRONMENT
);
$registry = Zend_Registry::getInstance();
$registry->configuration = $configuration;</code></pre>

Соответственно в контроллере получал значение следующим образом:

<pre><code class="php">$this->_nMaxFeeds = INTVAL(Zend_Registry::getInstance()
             ->configuration
             ->cms
             ->max
             ->feeds);</code></pre>
</blockquote>

Комментатор жалуется, что теперь он не знает как получить подобную функциональность в ZF1.8[^1]

[^1]: Видимо имеется в виду каноническое использование ZF, т.е. через Zend_Application и стандартный bootstraping.

Роб ответил:
<blockquote>
В контроллере можно сделать так:

<pre><code class="php">$bootstrap = $this->getInvokeArg('bootstrap');
$configArray = $bootstrap->getOptions();</code></pre>

А если нужен экземпляр объекта `Zend_Config`, то надо добавить строчку:

<pre><code class="php">$config = new Zend_Config($configArray);</code></pre>
</blockquote>

Такая вот маленькая хитрость, которая наверняка может сохранить массу времени при переходе на новый релиз.

__Кстати1:__ из этой заметки наверняка понятно, что вышел новый релиз ZF, но если кто еще не знает рекомендую ознакомиться с переводом анонса релиза ZF1.8 - [Вышел Zend Framework 1.8.0](http://zend-framework.ru/2009/05/zend-framework-1-8-0-reseas/ "Перевод анонса выпуска релиза ZF1.8")

__Кстати2:__ Рекомендую ознакомиться с [учебником Роба](http://akrabat.com/zend-framework-tutorial/ "Учебник / быстрый старт по ZF"), чтобы иметь представление о применении консоли ZF, если кто еще не пробовал её в работе.

__Upd:__ Там же в комментариях Роб предложил универсальное решение в "старом" стиле:

<pre><code class="php">class Bootstrap extends Zend_Application_Bootstrap_Base
{
   public function run()
   {
       Zend_Registry::set('config',
           new Zend_Config($this->getOptions()));
       parent::run();
   }
}</code></pre>
В этом случае объект конфига будет доступен в любом месте приложения.