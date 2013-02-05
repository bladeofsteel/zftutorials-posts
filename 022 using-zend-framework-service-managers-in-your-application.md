Использование сервис-менеджеров Zend Framework в ваших приложениях
-----------------------------------------


-----------------------------------------
__Источник:__ [Using Zend Framework service managers in your application][1]  
__Автор:__ [Jurian Sluiman][2]  
__Перевод:__ [Лобач Олег][3]  

Zend Framework 2 использует компонент _ServiceManager_ (сервис-менеджер, далее _СМ_) чтобы упростить применение инверсии управления[^1]. 

Замечу, что есть хорошие ресурсы об основах сервис-менеджеров (я рекомендую [этот пост Evan Coury][5] ([перевод][6]) или [этот пост Reese Wilson][7]), но многие люди все еще испытывают проблемы с настройкой _СМ_ под свои потребности. В этом посте я попытаюсь объяснить причины того, зачем фреймворк использует множество сервис-менеджеров, и возможность их использования в собственном коде. Я собираюсь осветить следующие темы:

1. Какие бывают сервис-менеджеры?
2. С какой целью использованы разные сервис-менеджеры?
3. Каким образом _ServiceLocator_ относится к _ServiceManager_?
4. Каким образом вы можете задать сервисы для всех тех сервис-менеджеров?
5. Как вы можете извлечь сервис из одного сервис-менеджера, находящегося внутри другого?

_СМ_ используется в различных местах Zend Framework 2, но эти четыре наиболее важные:

1. Основные сервисы приложения ("корневой сервис-менеджер" или "главный сервис-менеджер").
2. Контроллеры
3. Плагины контроллеров
4. Помощники вида

Каждая группа сервисов имеет собственный _СМ_, что дает возможность иметь один ключ для разных сервисов. Возможно вы знаете, что есть помощник вида "url", но так же есть и плагин контроллера "url". Этого было бы очень сложно достичь если у вас есть только один _СМ_ в который нужно добавить ключ "url", зависящий от контекста. С несколькими _СМ_ вы можете с легкостью иметь оба одновременно.

Существует так же момент безопасности. У вас может быть маршрут в котором есть параметр для контроллера. Набрав специальный URL, сервис-менеджер попытается инициализировать этот сервис для вас. Если вы не озаботитесь безопасностью, вы сможете случайно инициализировать любой объект, послав запрос на специальный URL.

## Отличия между _ServiceManager_ и _ServiceLocator_

Многие люди задают вопрос о различиях между менеджером и локатором. Сервис-локатор (далее _СЛ_) представляет собой очень "тонкий" интерфейс.

<pre class="lang:php"><code>
namespace Zend\ServiceManager;

interface ServiceLocatorInterface
{
    public function get($name);
    public function has($name);
}
</code></pre>

Сервис-менеджер это реализация сервис-локатора. По умолчанию в Zend Framework 2 реализацией сервис-локатора является сервис-менеджер. Во многих местах фреймворка вы иногда видите метод `getServiceLocator()`, а иногда метод `getServiceManager()`. Для метода `getServiceLocator()` вы на выходе получите _СЛ_, а для `getServiceManager()` вы явно запрашиваете реализацию _СМ_.

В настоящий момент большой разницы нет, так как оба метода возвращают один и тот же объект. Тем не менее вы можете решить использовать другую реализацию _СЛ_. Вы сами придерживаетесь договоренности о _СЛ_, но некоторые компоненты ZF2 все еще нуждаются в специфичной реализации _СМ_.

## Конфигурирование сервис-менеджера.

Сервис-менеджер может быть сконфигурирован двумя путями: класс модуля может вернуть конфигурацию _СМ_ и файл конфигурации модуля (`config/module.config.php` в большинстве случаев) может вернуть конфигурацию _СМ_. Результат обоих способов идентичен, так что использование того или иного, - лишь дело вкуса.

Вы можете добавить сервис любым из следующих способов:

<pre class="lang:php"><code>
/**
 * With the module class
 */
namespace MyModule;  

class Module
{
    public function getServiceConfig()
    {
        return array(
            'invokables' => array(
                'my-foo' => 'MyModule\Foo\Bar',
            ),
        );
    }
}
</code></pre>

<pre class="lang:php"><code>
/**
 * With the module config
 */
return array(
    'service_manager' => array(
        'invokables' => array(
            'my-foo' => 'MyModule\Foo\Bar'
        ),
    ),
);
</code></pre>

Как видите, в обоих методах содержимое массива одинаковое. Это справедливо для всех четырех типов сервис менеджеров. Для метода с классом модуля вы можете использовать "[утинную типизацию][8]"[^2] метода и конфигурация будет загружена[^3]. Так же вы можете "играть по правилам" и добавить интерфейс, сделав декларацию метода более строгой. С применением интерфейса ваш класс модуля будет выглядеть приблизительно так:

<pre class="lang:php"><code>
namespace MyModule;

use Zend\ModuleManager\Feature\ServiceProviderInterface;

class Module implements ServiceProviderInterface
{
    public function getServiceConfig()
    {
        return array(
            'invokables' => array(
                'my-foo' => 'MyModule\Foo\Bar',
            ),
        );
    }
}
</code></pre>

Для всех четырех сервис-менеджеров вы можете добавить ключ в ваш конфиг модуля или добавить метод в класс модуля. Позже вы можете выбрать утинную типизацию метода либо добавить интерфейс `Zend\ModuleManager\Feature\*`. Таблица, расположенная ниже, показывает связь между этими способами.

| Параметр | Значение |
|----------|----------|
| **_Сервисы приложения_** ||
| Manager | Application services |
| Manager class| Zend\ServiceManager\ServiceManager |
| Config key | service_manager |
| Module method | getServiceConfig() |
| Module interface | ServiceProviderInterface |
| **_Контроллеры_** ||
| Manager | Controllers |
| Manager class | Zend\Mvc\Controller\ControllerManager |
| Config key | controllers |
| Module method | getControllerConfig() |
| Module interface | ControllerProviderInterface |
| Service name | ControllerLoader |
| **_Плагины контроллера_** ||
| Manager | Controller plugins |
| Manager class | Zend\Mvc\Controller\PluginManager |
| Config key | controller_plugins |
| Module method | getControllerPluginConfig() |
| Module interface | ControllerPluginProviderInterface |
| Service name | ControllerPluginManager |
| **_Помощники вида_** ||
| Manager | View helpers |
| Manager class | Zend\View\HelperPluginManager |
| Config key | view_helpers |
| Module method | getViewHelperConfig() |
| Module interface | ViewHelperProviderInterface |
| Service name | ViewHelperManager |

  
где:

- Manager, - объект управления;
- Manager class, - класс сервис-менеджера;
- Config key, - наименование ключа, при использовании файла конфигурации;
- Module method, - название метода, при использовании метода класса модуля;
- Module interface, - интерфейс, который должен реализовывать класс модуля;
- Service name, - имя, под которым сервис будет зарегистрирован.

#### Будьте внимательны

Существует одна ловушка, о которой вам следует знать. Как [объяснил Evan][6], существует два параметра для фабрики. Вы можете использовать либо замыкание, либо строку, указывающую на класс. Этот класс должен реализовывать интерфейс `Zend\ServiceManager\FactoryInterface` либо должен иметь метод `__invoke`. Фабрики могут находиться внутри конфигурации модуля и в классе модуля.

Если у вас есть _замыкание_ **и** оно находится в `module.config.php`, значит у вас появилась проблема. Все настройки модулей могут быть закешированы как большой объединенный конфиг. Проблема с php в том, что _замыкание_ нельзя сериализовать. Таким образом, что бы воспользоваться замыканиями, вам нужно использовать либо фабрику классов в `module.config.php`, либо метод `getServiceConfig()`.

## "Корневой" менеджер vs остальные

Название "корневой" (или "основной") часто используется в обсуждениях, к примеру, по IRC, но в действительности никак не связано с наименованием какого либо кода в Zend Framework 2. Название, вероятно, происходит из идеи, что `Zend\ServiceManager\ServiceManager` содержит все основные сервисы, а остальные менеджеры более специализированны на конкретных типах сервисов. Название "корневой" предполагает, что есть связь между некоторыми менеджерами. И знаете что? Она, таки, есть!

Представьте, что у вас есть контроллер, в который вы хотите внедрить экземпляр хранилища кэша. У контроллера есть фабрика в _СМ_ контроллеров. Кэш - это сервис в основном _СМ_. Как вы собираетесь получить сервис кэша внутри фабрики контроллера? Вот тут и появляется на сцене _связь_. _СМ_ контроллеров, плагинов контроллеров и помощников видов наследуются от `Zend\ServiceManager\AbstractPluginManager`. Этот класс имеет метод `getServiceLocator()` который возвращает основной сервис-менеджер. Это дает возможность перемещаться между различными менеджерами:

<pre class="lang:php"><code>
use MyModule\Controller;

return array(
    'controllers' => array(
        'factories' => array(
            'MyModule\Controller\Foo' => function($sm) {
                $controller = new Controller\FooController;

                $cache = $sm->getServiceLocator()->get('my-cache');
                $controller->setCache($cache);

                return $controller;
            },
        ),
    ),
);
</code></pre>

В данном случае сервис кэша находится в основном сервис-менеджере и с помощью `$sm->getServiceLocator()` вы можете получить его.

Это становится еще более увлекательным, если вы узнаете, что менеджер плагинов контроллера и менеджер помощников представления зарегистрированы в качестве услуг в корневом _СМ_. Если есть сервис, который во время работы приложения нужно внедрить в помощник представления, вы легко можете сделать это. Например, помощник `url` содержит внедренный маршрутизатор, который нужен для сборки адреса по имени маршрута.

Вы можете получить из основного _СМ_ менеджер плагинов контроллера по ключу `"ControllerPluginManager"`. Менеджер помощников представления зарегистрирован под именем `"ViewHelperManager"`. Вы можете получить плагин примерно так:

<pre class="lang:php"><code>
use MyModule\Service;

return array(
    'service_manager' => array(
        'factories' => array(
            'MyModule\Service\Foo' => function($sm) {
                $service = new Service\Foo;

                $plugins = $sm->get('ViewHelperManager');
                $plugin  = $plugins->plugin('my-plugin');
                $service->setPlugin($plugin);

                return $service;
            },
        ),
    ),
);
</code></pre>

## Соседние сервис-менеджеры

Концепция соседних _СМ_ достаточно проста для понимания. Это способ сервис-менеджерам плагинов контроллера и помощников представления загрузить сервис из основного _СМ_ без использования `$sm->getServiceLocator()`. В этом и заключается идея "соседства", которая означает, что сервис-менеджер пытается загрузить сервис из основного _СМ_, если не смог загрузить его самостоятельно.

Так что, если вы посмотрите на пример выше, вы можете в некоторых случаях опустить вызов метода `getServiceLocator()` и загрузить сервис напрямую. Это справедливо **только для плагинов контроллера и помощников представления**. Причина очевидна. Это безопасность сервис-менеджера контроллеров: вы можете случайно создать экземпляр объекта, обратившись по специальному URL. Вы полностью рушите этот барьер когда разрешаете сервис-менеджеру контроллеров воспользоваться механизмом "соседства". Тем не менее, для плагинов контроллера и помощников представления все еще возможно использовать "соседство":

<pre class="lang:php"><code>
use MyModule\Controller\Plugin;

return array(
    'controller_plugins' => array(
        'factories' => array(
            'MyModule\Controller\Plugin\Foo' => function($sm) {
                $plugin = new Plugin\Foo;

                $cache = $sm->get('my-cache');
                $plugin->setCache($cache);

                return $plugin;
            },
        ),
    ),
);
</code></pre>

Выгода в том, что вы можете просто опускать `getServiceLocator()` для плагинов и помощников. Это, возможно, сделает ваш код более читаемым. Но читайте между строк: соседство не легко для непосредственного понимания. В предыдущем примере, `$sm` не содержит сервис `my-cache`, но если вы попробуете извлечь его, вы его получите. Очень хорошо документируйте такого рода фабрики, в противном случае вы можете столкнуться с проблемами в будущем!

## Личные предпочтения

По моему личному мнению, в своих модулях я предпочитаю строго следовать интерфейсам. Я всегда использую интерфейсы `Zend\ModuleManager\Feature`. Мне так же нравится стиль, когда все сервисы собраны в один файл конфигурации вместе с замыканиями и фабриками. Это позволяет пробежаться по всем ключам сервисов одного модуля без помех в виде конфигурации маршрутов (из настроек модуля) или настроек автозагрузки и логики бутстрапа (из класса модуля).

Обычно у меня есть кроме `module.config.php` еще и `service.config.php` в каталоге `config/`. И я подключаю этот файл просто как настройки модуля. Класс модуля зачастую выглядит примерно так:

<pre class="lang:php"><code>
namespace MyModule;

use Zend\Loader;
use Zend\ModuleManager\Feature;
use Zend\EventManager\EventInterface;

class Module implements
    Feature\AutoloaderProviderInterface,
    Feature\ConfigProviderInterface,
    Feature\ServiceProviderInterface,
    Feature\BootstrapListenerInterface
{
    public function getAutoloaderConfig()
    {
        return array(
            Loader\AutoloaderFactory::STANDARD_AUTOLOADER => array(
                Loader\StandardAutoloader::LOAD_NS => array(
                    __NAMESPACE__ => __DIR__ . '/src/' . __NAMESPACE__,
                ),
            ),
        );
    }

    public function getConfig()
    {
        return include __DIR__ . '/config/module.config.php';
    }

    public function getServiceConfig()
    {
        return include __DIR__ . '/config/service.config.php';
    }

    public function onBootstrap(EventInterface $e)
    {
        // Some logic
    }
}
</code></pre>

В `module.config.php` я предоставляю обычные настройки, а в `service.config.php` собраны вместе все сервисы. Пример такого типа установки показан в [EnsembleKernel][10], где `service.config.php` [выгдядит примерно так][11]. Но, конечно, существует масса других вариантов, вы можете выбрать любой по вашему вкусу.

[^1]: [Inversion of Control][4]
[^2]: [duck type][9]
[^3]: на сколько я понял, автор имеет в виду, что в классе модуля для загрузки конфига достаточно реализовать метод `getServiceConfig()`, — прим.пер.

[1]: http://juriansluiman.nl/en/article/120/using-zend-framework-service-managers-in-your-2
[2]: http://juriansluiman.nl/en/about
[3]: http://lobach.info/
[4]: http://en.wikipedia.org/wiki/Inversion_of_control
[5]: http://blog.evan.pro/introduction-to-the-zend-framework-2-servicemanager
[6]: /blog/introduction-to-the-zend-framework-2-servicemanager.html
[7]: http://zendblog.shinymayhem.com/2012/09/using-servicemanager-as-inversion-of.html
[8]: http://ru.wikipedia.org/wiki/%D3%F2%E8%ED%E0%FF_%F2%E8%EF%E8%E7%E0%F6%E8%FF
[9]: http://en.wikipedia.org/wiki/Duck_typing
[10]: https://github.com/ensemble/EnsembleKernel
[11]: https://github.com/ensemble/EnsembleKernel/blob/master/config/services.config.php