__Источник:__ [How to replace the “Action” helper in ZF 2 (and make great widgetized content)][5]  
__Автор:__ [Michaël Gallego][6]  
__Перевод:__ [Лобач Олег][7]  

Вчера у меня с Робертом Бейсиком ([Robert Basic][1]) было обширное обсуждение помощника вида "[Action][3]", пропавшего в Zend Framework 2. И так как многие спрашивают об этом, я решил написать на эту тему в блог. Я спрашивал его об этом потому, что ZfcTwig (официальный ZF2 модуль для [Twig][2], это потрясающий шаблонный движок от Fabien Potencier - попробуйте его и он сделает ваши шаблоны очень сексуальными) содержит встроенный помощник `Action`, но я чувствовал, что что-то с ним не так.

## Как это работает в ZF 1

Возвращаясь к ZF 1, помощник `action` в основном использовался для создания "виджетов", и позволял запустить из шаблона новый процесс диспетчеризации для другого действия. Это работало примерно так:

<pre lang="php"><code><?php echo $this->action('list', 'comment', null, array('count' => 10)); ?></code></pre>

Этот код просто вызывает действие `list` контроллера `CommentController` с некоторыми параметрами, и возвращает HTML-код. Легко и просто, не так ли?

Однако, этот метод страдает от [ряда недостатков][4]. Прежде всего это производительность. Из-за того, что он начинает новый MVC-процесс (диспетчеризация, маршрутизация&#8230;). Конечно же, все обработчики (hooks) событий (ACL&#8230;), которые вы могли добавить, будут снова запущены. Это может негативно повлиять на трудоемкость отладки, а так же критично снизить производительность.

Вторая причина заключается в том, что это приводит к зависимости вашего шаблона от контроллера. В принципе, при использовании помощника вида `action` запрос выполняется по следующей схеме:

    controler -> view -> controler -> view

В-третьих, когда дело касается виджетов, вам нужны действия[^1] (и, следовательно, маршруты) на каждое действие, которое вы можете вызвать в помощнике `action`. Это значит, что вам придется создать массу бессмысленных действий ("бессмысленные" потому, что они никогда не будут вызваны сами по себе) и массу бесполезных маршрутов[^2].

[^1]: **Прим. пер.:** _controller actions_
[^2]: **Прим. пер.:** _тут я с автором не согласен - можно обойтись без создания правила маршрутизации на каждый виджет, ведь при вызове помощника указывается действие, контроллер и модуль, а не маршрут._

Короче говоря, это **плохо** (ходят слухи, что Matthew Weier O&#8217;Phinney действительно считает его злом ;-)). В следствие этого, помощник `Action` был полностью удален из ZF 2.

## Решение

Ну, конечно, есть решения (опять же, благодарю Роберта Бейсика за то, что указал мне на одно из них!).

### Решение с помощью Forward

Первое решение - это использование плагина котроллера `Forward`. Плагин `forward` позволяет вам внутри одного действия направить поток исполнения приложения в другое действие, да еще и без накладных расходов (маршрутизация не выполняется). Например, допустим, что внутри конкретной страницы, вы хотите показать счет-фактуру. Потому, что счет-фактуру можно генерировать отдельно, и имеет смысл вынести эту задачу в другой контроллер, вы можете сделать так:

<pre lang="php"><code>
<?php
public function invoiceDetailsAction()
{
    $mainViewModel = new ViewModel();

    [...] // Выполняем необходимы действия, устанавливаем переменные для этой страницы...

    // Получим виджет для счета-фактуры
    $invoiceId     = $this->params('id');
    $invoiceWidget = $this->forward()->dispatch('Application\Controller\Invoice', array(
        'action' => 'display',
        'id'     => $invoiceId
    ));

    return $mainViewModel->addChild($invoiceWidget, 'invoiceWidget');
}
?>
</code></pre>

Это действие сначала создает экземпляр `ViewModel` (модели представления) для текущей страницы (действие invoice-details). Вы можете получить и передать в представление некоторые детали, специфичные для этой страницы.

Затем, я вызываю плагин контроллера `forward`:

<pre lang="php"><code>
    $invoiceWidget = $this->forward()->dispatch('Application\Controller\Invoice', array(
        'action' => 'display',
        'id'     => $invoiceId
    ));
</code></pre>

Первый параметр - название контроллера, второй - массив параметров. В большинстве случаев, вы будете устанавливать ключ "action" (и в следствие этого, вам не нужно явно добавлять маршрут, плагин `forward` уже имеет все необходимое для обработки запроса).

Плагин `forward` возвращает экземпляр `ViewModel`. Наконец, мы добавляем эту модель представления, в качестве дочерней, к основной модели представления. А в вашем шаблоне вам просто нужно сделать следующее:

<pre lang="php"><code>
<?php echo $this->invoiceWidget; ?>
</code></pre>

Легко и просто. Ваш код хорошо разделен, он не страдает от проблем с производительностью и не содержит ссылок на контроллеры внутри шаблона.

### Метод с помощником вида

Но этот метод страдает от большого недостатка: он прекрасен, когда ваш виджет выводится на однй странице, ну, максимум на двух.

Но что если вы хотите выводить метео-виджет на всех ваших страницах? Конечно, вы не хотите использовать `forward` в КАЖДОМ действии. Это было бы подвержено ошибкам, сделало бы ваш код менее поддерживаемым. Для этого конкретного случая, помощник `action` смотрелся бы отлично, как если бы вы могли сделать в вашем шаблоне так:

<pre lang="php"><code>
<?php echo $this->action('displayWidget', 'Meteo', null, array('city' => 'Paris')); ?>
</code></pre>

К счастью, в Zend Framework 2 есть более удачная альтернатива: помощники вида. Идея заключается в создании помощника вида для каждого "виджета". Ваш помощник вида будет получать данные, например, из базы данных (в этом случае, вы должны работать с зависимостями через ServiceManager, как - будет показано позже) или от веб-сервиса. После того, как он получит данные, он будет вызывать помощник `Partial` (или самостоятельно генерировать HTML непостредственно в помощнике вида) и возвращать сгенерированный HTML.

Давайте останемся с нашим метео-примером и создадим простой помощник вида для него. Наш помощник вида будет зависеть от сервиса `MeteoService`:

<pre lang="php"><code>
<?php
namespace Application\View\Helper;

use Application\Service\MeteoService;
use Zend\View\Helper\AbstractHelper;

class MeteoWidget extends AbstractHelper
{
    protected $meteoService = null;
    
    public function __construct(MeteoService $meteoService)
    {
        $this->meteoService = $meteoService;
    }

    public function __invoke($city)
    {
        $temperature = $this->service->getTemperature($city);

        return $this->getView()->partial('application/meteo/display', array('temperature' => $temperature));

        // If a full template is overkill, you could of course just render
        // the widget directly
        return "<div>The temperature is $temperature degrees</div>";
    }
}
?>
</code></pre>

Так как помощник вида имеет зависимость, вы должны сообщить `Service Manager` каким образом  обработать эту зависимость:

<pre lang="php"><code>
<?php
// In your Module.php

public function getViewHelperConfig()
{
    return array(
        'factories' => array(
            'meteoWidget' => function ($serviceManager) {
                // Get the Meteo Service
                $meteoService = $serviceManager->getServiceLocator()->get('MeteoService');
                return new \Application\View\Helper\MeteoWidget($meteoService);
            }
        )
    );
}
?>
</code></pre>

И в вашем шаблоне:

<pre lang="php"><code>
<?php echo $this->meteoWidget('Paris'); ?>
</code></pre>

И вуаля! У вас есть хорошая архитектура, поддерживающая виджеты, которые вы можете повторно использовать на любой странице!

[1]: http://robertbasic.com "Robert Basic's blog"
[2]: http://twig.sensiolabs.org "Twig"
[3]: http://framework.zend.com/manual/1.12/en/zend.view.helpers.html#zend.view.helpers.initial.action "Action View Helper"
[4]: http://www.rmauger.co.uk/2009/03/why-the-zend-framework-actionstack-is-evil/ "Why The Zend Framework Action Stack Is Evil"
[5]: http://www.michaelgallego.fr/blog/?p=223
[6]: http://www.michaelgallego.fr/blog/
[7]: http://lobach.info/