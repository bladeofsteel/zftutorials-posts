__Источник:__ [Introduction to the Zend Framework 2 ServiceManager][1]  
__Автор:__ [Evan Coury][2]  
__Перевод:__ [Лобач Олег][3]  

Эта статья предназначена для знакомства с новым компонентом Zend Framework 2 _ServiceManager_: описание различных особенностей наряду с некоторыми простыми примерами.

Итак, что такое _ServiceManager_? В основном это реестр, или контейнер (правильный термин [Service Locator][4]) для хранения различных объектов необходимых вашему приложению, что позволяет легко применять инверсию зависимостей ([Inversion of Control][5]). _ServiceManager_ содержит только ту информацию, которая необходима для отложенной инициализации (lazily instantiate) запрашиваемых объектов. Так что если вы думали о "сервисах" как о составляющей [сервисного слоя][6], то вам может быть лучше думать о _ServiceManager_ как о "менеджере объектов" или "менеджере экземпляров".

## Особенности Service Manager

На первый взгляд, компонент `Zend\ServiceManager` может показаться простым реестром или сервис локатором, и это так. Вы можете просто установить или получить объект по заданному имени:

<pre class="lang:php"><code>
    $serviceManager->setService('some_service', new SomeService());
    $someService = $serviceManager->get('some_service');
</code></pre>

[stextbox id="warning" caption="Примечание" collapsed=true]Все имена сервисов нечувствительны к регистру и из них удаляются следующие символы: `-`, `_`, `\`, `/`, а так же пробелы.[/stextbox]

Тем не менее, компонент _ServiceManager_ предоставляет некоторые дополнительные возможности, чтобы облегчить нам жизнь. Экземпляр _ServiceManager_ (далее _SM_) может содержать следующее:

### Invokables

`Invokables` это просто полное имя класса, представленное в _SM_ некой строкой. При запросе оно будет проинициализированно простым `new $invokableClass();`.

<pre class="lang:php"><code>
    $serviceManager->setInvokableClass('index', 'MyModule\Controller\IndexController');
    $indexController = $serviceManager->get('index');
</code></pre>

### Фабрики (Factories)

Фабрика это либо [PHP callable][7], либо объект, либо полное имя класса, реализующего интерфейс [Zend\ServiceManager\FactoryInterface][8]. Фабрики используются для любых настроек или [внедрения зависимостей][9] необходимых объекту в момент, когда объект запрашивается из _SM_.

<pre class="lang:php"><code>
    $serviceManager->setFactory('user_mapper', function($serviceManager) {
        $mapper    = new \MyModule\Mapper\User;
        $dbAdapter = $serviceManager->get('db_adapter');
        $mapper->setDbAdapter($dbAdapter);
        return $mapper;
    });
    $userMapper = $serviceManager->get('user_mapper');
</code></pre>

### Псевдонимы (Aliases)

Псевдонимы - это просто указатели с одного названия сервиса на другое и они могут быть рекурсивными. Это может показаться бессмысленным, но псевдонимы на самом деле играют очень важную роль в модульном окружении.

<pre class="lang:php"><code>
    $serviceManager->setAlias('usermodule_db_adapter', 'db_adapter');
    $db = $serviceManager->get('usermodule_db_adapter'); // На самом деле будет получен 'db_adapter'
</code></pre>

### Инициализаторы (Initializers)

Инициализатор - это либо [замыкание][10], либо объект, либо полное имя класса, реализующего [Zend\ServiceManager\InitilizerInterface][11]. Любой объект, вытаскиваемый из _SM_, проходит через зарегистрированные инициализаторы, которые могут выполнить дополнительные задачи по инициализации.

<pre class="lang:php"><code>
    $serviceManager->addInitilizer(function($instance, $serviceManager) {
        if ($instance instanceof SomethingAwareInterface) {
            $instance->setSomething($serviceManager->get('something'));
        }
    });
</code></pre>

### Конфигурационные классы (Configuration Classes)

Конфигурационный класс - это класс реализующий [Zend\ServiceManager\ConfigInterface]. Конфигурационные классы просто знают как настроить экземпляр _SM_, возможно установить несколько фабрик, инициализаторов, и пр.

<pre class="lang:php"><code>
    $serviceConfig = new ServiceConfig;
    $serviceConfig->configureServiceManager($serviceManager);
</code></pre>

### Разделяемые сервисы (Shared Services)

Все, что находится в _SM_, может быть общим или частным. Если общее, то _SM_ создает экземпляр запрашиваемого сервиса при первом запросе этого объекта, а при последующих запросах возвращает тот же самый объект. В противном случае _SM_ на каждый запрос будет создавать новый экземпляр сервиса. По умолчанию, все сервисы являются общими.

<pre class="lang:php"><code>
    $serviceManager->setInvokable('my_service', 'My\Service');
    $serviceA = $serviceManager->get('my_service');
    $serviceB = $serviceManager->get('my_service'); // Точно такой же экземпляр как $serviceA

    $serviceManager->setShared('my_service', false);
     
    $serviceC = $serviceManager->get('my_service'); // Новый экземпляр My\Service
     
    var_dump((spl_object_hash($serviceA) === spl_object_hash($serviceB))); // bool(true)
    var_dump((spl_object_hash($serviceB) === spl_object_hash($serviceC))); // bool(false)
</code></pre>

### Абстрактные фабрики (Abstract Factories)

Если у _SM_ запрашивают сервис, который он не может обнаружить, он станет опрашивать все зарегистрированные абстрактные фабрики, что бы посмотреть сможет ли какая-либо из них создать необходимый объект. Абстрактная фабрика может быть либо строкой, содержащей полное имя класса, либо объектом, реализующим интерфейс [Zend\ServiceManager\AbstractFactoryInterface][13].

[stextbox id="warning" caption="Примечание" collapsed=true]В стандартной конфигурации ZF2 MVC, если у вас есть какие-то DI-определения в конфигурации вашего приложения, в главном _ServiceManager_ будет зарегистрирована абстрактная фабрика DI. Это означает, что если запрашиваемый сервис отсутствует в _SM_, будет предпринята попытка извлечь его из контейнера `Zend\Di\Di`.[/stextbox]

### Соседство (Peering Service Managers)

_ServiceManager_ вводит понятие соседства. Каждый _ServiceManager_ может иметь "соседа", а точнее, стек из одного и более сервис-менеджеров, которые могут быть использованы когда сервис достается из _SM_.

Класс `ServiceManager` так же имеет возможность указать ему пытаться обратиться к соседним _SM_ прежде чем попытаться искать запрашиваемый сервис у себя. Это поведение может быть установлено посредством `$serviceManager->setRerieveFromPeeringManagerFirst(true);`. Помимо этого различия соседние _SM_ ведут себя аналогично абстрактным фабрикам.

<pre class="lang:php"><code>
    $serviceManagerA = new ServiceManager();
    $serviceManagerA->setInvokableClass('index', 'MyModule\Controller\IndexController');
     
    // Создать новый сервис-менеджер, который имеет $serviceManagerA в качестве соседнего.
    $serviceManagerB = $serviceManagerA->createScopedServiceManager(ServiceManager::SCOPE_PARENT);
    $indexController = $serviceManagerB->get('index'); // Работает!
     
    // Создать новый сервис-менеджер и добавить его как соседа в $servicemanagerA
    $serviceManagerC = $serviceManagerA->createScopedServiceManager(ServiceManager::SCOPE_CHILD);
    $indexController = $serviceManagerC->get('index'); // ServiceNotFoundException!
     
    $serviceManagerC->setInvokableClass('foo', 'MyModule\Foo');
    $foo = $serviceManagerA->get('foo'); // Работает!
</code></pre>

[stextbox id="warning" caption="Примечание" collapsed=true]По умолчанию, _SM_ “ControllerLoader” (который настраивается через ключ ‘controllers’ в конфигурации или `Module::getControllerConfiguration()` в ваших модулях) устанавливается с основным сервис-менеджером в качестве соседнего.[/stextbox]

[stextbox id="info"]Для дальнейшего изучения обращайтесь к [Service Manager Quick-Start][14][/stextbox]

[1]: http://blog.evan.pro/introduction-to-the-zend-framework-2-servicemanager
[2]: http://evan.pro/
[3]: http://lobach.info/
[4]: http://en.wikipedia.org/wiki/Service_locator_pattern
[5]: http://en.wikipedia.org/wiki/Inversion_of_control
[6]: http://martinfowler.com/eaaCatalog/serviceLayer.html
[7]: http://php.net/manual/en/language.types.callable.php
[8]: https://github.com/zendframework/zf2/blob/master/library/Zend/ServiceManager/FactoryInterface.php
[9]: http://martinfowler.com/articles/injection.html
[10]: http://php.net/manual/en/functions.anonymous.php
[11]: https://github.com/zendframework/zf2/blob/master/library/Zend/ServiceManager/InitializerInterface.php
[12]: https://github.com/zendframework/zf2/blob/master/library/Zend/ServiceManager/ConfigInterface.php
[13]: https://github.com/zendframework/zf2/blob/master/library/Zend/ServiceManager/AbstractFactoryInterface.php
[14]: http://zf2.readthedocs.org/en/latest/modules/zend.service-manager.quick-start.html