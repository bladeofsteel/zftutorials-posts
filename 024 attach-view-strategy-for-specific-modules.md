Прикрепление стратегии представления к конкретным модулям
-----------------------------------------
Простой пример использования в конкретном модуле отличной от применяемой в остальных частях приложения стратегии представления. Кроеме того, этот небольшой пост содержит пояснение механизма работы слоя представления в Zend Framework 2.
-----------------------------------------
__Источник:__ [Attach a view strategy for specific modules][1]  
__Автор:__ [Jurian Sluiman][2]  
__Перевод:__ [Лобач Олег][3]  

[1]: http://juriansluiman.nl/en/article/119/attach-a-view-strategy-for-specific-modules
[2]: http://juriansluiman.nl/en/about
[3]: http://lobach.info/

На канале IRC и в Twitter звучали вопросы о том, как можно включить _Json стратегию представления_ только для конкретного модуля в Zend Framework 2. Так как мне тоже в скором времени потребудется эта возможность, я начал исследовать код `Zend\View` и его стратегий.

Фактически, код очень прост, но как и у других, кто сражался с ним, он отнял некоторое время, прежде чем я получил результаты. Если вам не интересно, как это работает, просто скопируйте код, расположенный ниже, в ваш модуль и все должно заработать.

<pre class="lang:php"><code>
namespace MyModule;

use Zend\ModuleManager\Feature;
use Zend\EventManager\EventInterface;
use Zend\Mvc\MvcEvent;

class Module implements
    Feature\BootstrapListenerInterface
{
    public function onBootstrap(EventInterface $e)
    {
        $app = $e->getApplication();
        $em  = $app->getEventManager()->getSharedManager();
        $sm  = $app->getServiceManager();

        $em->attach(__NAMESPACE__, MvcEvent::EVENT_DISPATCH, function($e) use ($sm) {
            $strategy = $sm->get('ViewJsonStrategy');
            $view     = $sm->get('ViewManager')->getView();
            $strategy->attach($view->getEventManager());
        });
    }
}
</code></pre>

В этом примере, я внедряю `JsonStrategy` только когда контроллеры в моем модуле будут диспетчеризованы. Поэтому я "слушаю" событие `dispatch` только в мененджере событий, имеющим такой же идентификатор, как и пространство имен моего модуля. В этом случае я уверен, что каждый `MyModule\Controller\SomeController` станет причиной срабатывания события, но любой другой `AnotherModule\Controller\SomeController` таковым не будет.

_Стратегия_ извлекается из сервис-локатора, куда также непосредственно внедрён _Json-визуализатор_ (так что нам не нужно заботиться об этом). Сервис-менеджер так же содержит зарегистрированный `ViewManager`, который является `Zend\Mvc\View\Http\ViewManager`, соединяющий `MVC`-части фреймворка с компонентом `Zend\View`. Этот менеджер, очевидно, осведомлен о `Zend\View\ View`, отвечающим за визуализацию скриптов представления и тому подобном. Представление имеет собственный менеджер событий, вызывающий события для получения визуализатора и внедрения ответа в объект `Response`. Это то место, которое `JsonStrategy` хочет прослушивать, поэтому _стратегия_ прикрепляет несколько _слушателей_ к менеджеру событий представления.

Потребовалось некоторое время, чтобы понять это, но, надеюсь, другие извлекут выгоду из этой статьи и не застрянут в изучении MVC-слоев Zend Framework 2.