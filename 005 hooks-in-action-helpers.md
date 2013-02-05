__Источник:__ [Hooks in Action Helpers][1]
__Автор:__ [Rob Allen](http://akrabat.com/)
__Перевод:__ [Лобач Олег](http://lobach.info/)

Продолжая обсуждение [помощников действий Zend Framework][2] (мой перевод: [Использование помощников действий][3], — прим. пер.), давайте поговорим о перехватчиках в них.

 [1]: http://akrabat.com/zend-framework/hooks-in-action-helpers/
 [2]: http://akrabat.com/zend-framework/using-action-helpers-in-zend-framework/
 [3]: /blog/using-action-helpers.html

Перехватчики – это особенность помощников действий, которая позволяет вам выполнить некий код в определенных точках цикла диспетчеризации. Собственно, для помощников действий доступно всего два типа перехватчиков:

* `preDispatch()`: вызывается перед запуском действия
* `postDispatch()`: вызывается после завершения работы действия

Это позволяет вам быть уверенным в том, что некоторая функциональность всегда будет выполнена при каждом запросе. Рассмотрим простой пример вывода случайной цитаты в «подвале» сайта.

Мы начнем с помощника действия в нашем каталоге `controllers/helpers`, названного `Quote`:

<pre><code class="php"><?php

class Zend_Controller_Action_Helper_Quote extends Zend_Controller_Action_Helper_Abstract
{
    function preDispatch()
    {
        $view = $this->getActionController()->view;
        $view->footerQuote = $this->getQuote();
    }

    function getQuote()
    {
        $quotes[] = 'I want to run, I want to hide, I want to tear down the walls';
        $quotes[] = 'One man come in the name of love, One man come and go';
        return $quotes[rand(0, count($quotes)-1)];
    }
}</code></pre>

Метод `preDispatch()` получает объект представления из контроллера действия и присваивает случайную цитату свойству `footerQuote` этого объекта.

Мы должны сказать брокеру помощников, что мы хотим активировать этот перехватчик. Для этого в дополнение к вызову `addPath()`, наш загрузочный файл нужно дополнить вызовом `addHelper()`. После этого загрузчик станет содержать код:

<pre><code class="php">    // Action Helpers
    Zend_Controller_Action_HelperBroker::addPath(
        APPLICATION_PATH .'/controllers/helpers');

    $hooks = Zend_Controller_Action_HelperBroker::getStaticHelper('Quote');
    Zend_Controller_Action_HelperBroker::addHelper($hooks);</code></pre>

Пока мы используем `addPath()` для указания брокеру помощников где искать помощники действий, мы можем использовать `getStaticHelper()` в качестве простого способа инстанцирования класса без `require()` и последующего вызова `new`. Затем мы можем зарегистрировать его с помощью помощника брокера использовав `addHelper()`.

Так как цитата отображается в подвале сайта, требуется внести изменения в HTML-код в `layout.phtml`:

<pre><code class="html"><div id="footer">
    <div id="quote">
        <?php echo $this->footerQuote; ?>
    </div>
</div></code></pre>

Вот и все - не сложно, правда?