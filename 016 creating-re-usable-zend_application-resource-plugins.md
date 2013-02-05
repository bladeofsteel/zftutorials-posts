__Источник:__ [Creating Re-Usable Zend_Application Resource Plugins](http://weierophinney.net/matthew/archives/231-Creating-Re-Usable-Zend_Application-Resource-Plugins.html)  
__Автор:__ [Matthew Weier O'Phinney](http://weierophinney.net/matthew/)  
__Перевод:__ [Лобач Олег](http://lobach.info/)

В моей [последней статье](http://weierophinney.net/matthew/archives/230-Quick-Start-to-Zend_Application_Bootstrap.html) ("[Быстрый старт с Zend_Application_Bootstrap](/blog/quick-start-to-zend_application_bootstrap.html)") я писал о том, как начать работу с `Zend_Application`, в том числе информацию о том, как писать методы ресурсов, а также список доступных плагинов ресурсов. Что произойдет, когда вам потребуется повторно используемый ресурс, для которого не поставляется готовый плагин? Ответ прост: напишите ваш собственный, конечно!

Все плагины в `Zend Framework` следуют [общему шаблону](http://framework.zend.com/manual/en/learning.plugins.intro.html). В общем случае, вы группируете плагины в общем каталоге, с общим префиксом класса, а затем уведомляете подключающий класс о их местонахождении.

В этой статье давайте предположим, что вы хотите, чтобы ресурс плагина выполнял следующие действия:

- Установил DOCTYPE представления (`View`)
- Задал заголовок страницы и разделитель по умолчанию

## Начало работы {#part1}

Прежде всего, давайте определим префикс класса, который мы будем использовать. Если мы будем следовать [Стандартам кодирования Zend Framework](http://framework.zend.com/manual/en/coding-standard.overview.html), мы сможем эффективно использовать автозагрузку, при одновременном обеспечении общего префикса класса для наших ресурсов.

Для целей данного упражнения, мы будем использовать префикс класса `Phly_Resource`, расположенного в `Phly/Resource/` нашего `include_path`.

Назовем наш особый ресурс `Layouthelpers`, с полным названием класса `Phly_Resource_Layouthelpers`, и поместим его в `Phly/Resource/Layouthelpers.php`. Он должен реализовывать `Zend_Application_Resource_Resource`, но зачастую бывает проще расширить `Zend_Application_Resource_ResourceAbstract`. В обоих случаях необходимо определить метод `init()`. Давайте реализуем скелет нашего класса:

<pre lang="php"><code>
<?php
// Phly/Resource/Layouthelpers.php
//
class Phly_Resource_Layouthelpers 
    extends Zend_Application_Resource_ResourceAbstract
{
    public function init()
    {
    }
}</code></pre>

## Об отслеживании зависимостей {#part2}

В моей предыдущей статье я показал пример отслеживания зависимостей в `Zend_Application`. Нам необходимо будет этим воспользоваться, так как обе наши задачи взаимодействуют с объектом представления, который мы будем извлекать с помощью ресурса `View`.

При создании методов ресурсов непосредственно в вашем загрузке, вы можете просто вызвать `$this->getResource($name)`. Однако, в классе плагина ресурсов, необходимо сначала получить доступ к самому объекту начальной загрузки - с помощью метода `getBootstrap()`.

Давайте убедимся в том, что ресурс представления инициализирован, и извлечем его.

<pre lang="php"><code class="php">
<?php
// Phly/Resource/Layouthelpers.php
//
class Phly_Resource_Layouthelpers 
    extends Zend_Application_Resource_ResourceAbstract
{
    public function init()
    {
        $bootstrap = $this->getBootstrap();
        $bootstrap->bootstrap('View');
        $view = $bootstrap->getResource('View');

        // ...
    }
}</code></pre>

## Настройка ресурса {#part3}

Теперь, когда мы получили наш объект представления, мы можем сделать определенную работу. Так как мы хотим, чтобы ресурс можно было повторно использовать, мы должны разрешить некоторые параметры конфигурации. `Zend_Application_Resource_ResourceAbstract` предоставляет некоторую шаблонную функциональность для этого.

Во-первых, мы предоставим некоторые параметры по умолчанию через свойство `$_options`.

<pre lang="php"><code class="php">
<?php
// Phly/Resource/Layouthelpers.php
//
class Phly_Resource_Layouthelpers 
    extends Zend_Application_Resource_ResourceAbstract
{
    protected $_options = array(
        'doctype'         => 'XHTML1_STRICT',
        'title'           => 'Site Title',
        'title_separator' => ' :: ',
    );

    public function init()
    {
        $bootstrap = $this->getBootstrap();
        $bootstrap->bootstrap('View');
        $view = $bootstrap->getResource('View');

        // ...
    }
}</code></pre>

Затем мы можем получить параметры воспользовавшись методом `getOptions()`.

<pre lang="php"><code class="php">
<?php
// Phly/Resource/Layouthelpers.php
//
class Phly_Resource_Layouthelpers 
    extends Zend_Application_Resource_ResourceAbstract
{
    protected $_options = array(
        'doctype'         => 'XHTML1_STRICT',
        'title'           => 'Site Title',
        'title_separator' => ' :: ',
    );

    public function init()
    {
        $bootstrap = $this->getBootstrap();
        $bootstrap->bootstrap('View');
        $view = $bootstrap->getResource('View');

        $options = $this->getOptions();
        // ...
    }
}</code></pre>

Теперь, в файлах конфигурации разработчики могут изменять стандартные значения:

<pre class="nums:false lang:default highlight:0 decode:true "><code class="ini">[production]
; ...
resources.layouthelpers.doctype = "HTML5"
resources.layouthelpers.title = "My Snazzy New Website"
resources.layouthelpers.title_separator = " &emdash; "</code></pre>

## Выполняем некоторую работу {#part4}

Теперь, когда у нас есть все нужные части, давайте сделаем основную работу:

<pre lang="php"><code class="php">
<?php
// Phly/Resource/Layouthelpers.php
//
class Phly_Resource_Layouthelpers 
    extends Zend_Application_Resource_ResourceAbstract
{
    protected $_options = array(
        'doctype'         => 'XHTML1_STRICT',
        'title'           => 'Site Title',
        'title_separator' => ' :: ',
    );

    public function init()
    {
        $bootstrap = $this->getBootstrap();
        $bootstrap->bootstrap('View');
        $view = $bootstrap->getResource('View');

        $options = $this->getOptions();
        
        $view->doctype($options['doctype']);
        $view->headTitle()->setSeparator($options['title_separator'])
                          ->append($options['title']);
    }
}</code></pre>

Это все!

## Расскажем загрузчику о нас {#part5}

Ну, это все что нужно сделать для реализации плагина ресурса. Но как мы расскажем нашему классу загрузчика о нем? Через наш конфигурационный файл, используя ключ `pluginPaths`. Это массив, ключами которого являются префиксы классов плагинов, а значениями - путь, соответствующий префиксу.

<pre class="nums:false lang:default highlight:0 decode:true "><code class="ini">[production]
; ...
pluginPaths.Phly_Resource = "Phly/Resource"
resources.layouthelpers.doctype = "HTML5"
resources.layouthelpers.title = "My Snazzy New Website"
resources.layouthelpers.title_separator = " &emdash; "</code></pre>

Вы можете зарегистрироваться так много путей к плагинам, сколько захотите. Так как этот ключ обрабатывается до исполнения любого из ресурсов, он может быть определен в любом месте вашего конфигурационного файла.

## Дальнейшие соображения {#part6}

Пример из этого поста тривиален. Но один аспект не был обсужден - создание ресурса, который будет использоваться повсюду в вашем приложении. Например, вы можете захотеть создать ресурс, который вы будете использовать в разное время в вашем приложении. Если вы возвращаете значение из вашего метода `init()`, объект начальной загрузки сохранит его для последующего извлечения. Хороший пример этого мы видели раньше: ресурс `View` зарегистрировал объект `Zend_View` в классе начальной загрузки просто возвратив экземпляр из плагина ресурса.

## Выводы {#part7}

Надеюсь, эта и предыдущая статьи помогли пролить некоторый свет на `Zend_Application` и, в частности, как писать и загружать ресурсы.

Если у Вас есть дополнительные вопросы, вы можете найти меня в [списке рассылки ZF](http://framework.zend.com/archives), на IRC через серверы Freenode, или в [Twitter](http://twitter.com/weierophinney). Удачи!