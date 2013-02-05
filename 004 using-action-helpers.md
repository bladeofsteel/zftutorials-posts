__Источник:__ [Using Action Helpers in Zend Framework][2]
__Автор:__ [Rob Allen][1]
__Перевод:__ [Лобач Олег](http://lobach.info/)

 [1]: http://akrabat.com/
 [2]: http://akrabat.com/zend-framework/using-action-helpers-in-zend-framework/

Когда вам нужно использовать один и тот же функционал в нескольких контроллерах, можно воспользоваться помощником действий. Помощники действий являются очень мощным инструментом и содержат механизмы автоматического запуска, когда вы в них нуждаетесь, но вы можете проигнорировать все, в случае если Вам это не нужно.

Первое, что вы должны сделать - решить где разместить их. Последняя версия рекомендаций по стандартной структуре проекта предлагает использовать подкаталоги вашего каталога контроллеров. Т.е. **`application/controllers/helpers/`**, и это лучший вариант.

Во-первых, вы должны сказать брукеру помощников где располагаются ваши помощники действий. Я обычно делаю это в загрузчике (bootstrap file), но точно так же можно сделать это и в плагине фронт-контроллера.

<pre><code class="php">
    Zend_Controller_Action_HelperBroker::addPath(
        APPLICATION_PATH .'/controllers/helpers'
    );
</code></pre>

Затем вы должны создать помощник действий, мы назовем его **Multiples** в нашем примере, тогда название файла будет `application/controllers/helpers/Multiples.php`:

<pre><code class="php"><?php

class Zend_Controller_Action_Helper_Multiples extends
                Zend_Controller_Action_Helper_Abstract
{
    function direct($a)
    {
        return $a * 2;
    }
}</code></pre>

Обратите внимание на префикс названия класса помощника действий - `Zend_Controller_Action_Helper`. Вы можете поменять этот префикс на другой, передав его вторым параметром в функции `Zend_Controller_Action_HelperBroker::addPath()`, вызвав её в вашем загрузчике.

Наконец, использование в действии контроллера:

<pre><code class="php"><?php

class IndexController extends Zend_Controller_Action
{
    public function indexAction()
    {
        $this->view->headTitle('Home');
        $this->view->title = 'Test of the Multiples action helper';

        $number = 30;
        $twice = $this->_helper->multiples($number);

        $this->view->number = $number;
        $this->view-twice = $twice;
    }
}</code></pre>

Обратите внимание на то, что мы вызываем помощник действия как метод класса `_helper`. Этот вызов транслируется в вызов метода `direct()` нашего класса помощника.

Также вы можете написать несколько методов в одном помощнике действий:

<pre><code class="php">class Zend_Controller_Action_Helper_Multiples extends
                Zend_Controller_Action_Helper_Abstract
{
    function direct($a)
    {
        return $a * 2;
    }

    function thrice($a)
    {
        return $a * 3;
    }
}</code></pre>

Для вызова метода `thrice()` в действии вашего контроллера делайте так:

<pre><code class="php">$thrice = $this->_helper->multiples->thrice($number);</code></pre>

Таким образом, если вы используете имя помощника действий как свойство `_helper`, то вы можете вызвать любой метод из этого помощника.

Это всего лишь общие сведения для начала освоения. За подробностями обращайтесь к руководству.

Удачи и избегайте копи-паста общей функциональности!