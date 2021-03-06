Интерфейсные инъекции через инициализаторы в Zend\ServiceManager
-----------------------------------------
Что такое интерфейсные инъекции, чем отличаются "мягкие" и "жесткие" зависимости, как Zend\ServiceManager помогает в реализации интерфейсных инъекций? Ответы на эти вопросы, а так же пример практической задачи применения интерфейсных инъекций, содержатся в переводе статьи Jurian Sluiman "Interface injection with initializers in Zend\ServiceManager".

-----------------------------------------
__Источник:__ [Interface injection with initializers in Zend\ServiceManager][1]  
__Автор:__ [Jurian Sluiman][2]  
__Перевод:__ [Лобач Олег][3]  

[1]: http://juriansluiman.nl/en/article/121/interface-injection-with-initializers-in-zend-servicemanager
[2]: http://juriansluiman.nl/en/about
[3]: http://lobach.info/

`Zend\ServiceManager` является компонентом, который занимается (помимо других вещей) внедрением зависимостей. Во время разработки Zend Framework 2 я тщательно посмотрел на внедрение зависимостей. Обратите внимание на  хороший ресурс от Мартина Фаулера, где он объясняет существующие [три типа внедрения зависимостей][4]. В этом посте мне особенно интересны инъекции "мягких" зависимостей через интерфейсные инъекции.

[4]: http://www.martinfowler.com/articles/injection.html#FormsOfDependencyInjection

Помощь `Zend\ServiceManager` заключается в том, что иметь разделенные (не связанные) объекты становится просто. Для использования тех случаях, когда у вас есть "мягкие" зависимости, `ServiceManager` имеет отличный инструмент под названием инициализаторы. Инициализаторы - это небольшой "вызываемый код", обеспечивающий дополнительную функциональность вашим объектам при извлечении их из сервис-менеджера.

## "Мягкие" зависимости

В качестве примера я буду использовать класс логгера. Объект `Zend\Log\Logger` может записывать сообщения в так называемых "писателей", логгеры мне видятся типичной мягкой зависимостью. Помимо определения зависимости я бы сказал, что жесткая зависимость является «максимальным ограничением для службы для выполнения заданной задачи и непосредственно связана с характером цели". Мягкая зависимость напротив является "минимальным ограничением для службы, избирательно используется в выполнении задачи и не связана с характером цели".

На практике это означает, что если у вас есть сервис `BloggerService` и метод `createPost()`, у вас есть жесткая зависимость от адаптера базы данных, если вы сохраняете ваши сообщения в блоге в базе данных: это требуется для выполнения задачи и база данных связана с целью сохранить сообщение. Если вы по желанию отправляете твит о том, что вы опубликовали в блоге что-то новое, вы можете внедрить `TweetService` в `BloggerService`. Если у `BloggerService` есть `TweetService`, он отправит твит. Если вы не установили `TweetService`, он не будет твитить: сервис отправки твитов не является необходимым для выполнения основной задачи и, кроме того, не связан с характером сохранения сообщения блога.

## Инъекция интерфейса

Если вы хотите внедрить мягкие зависимости с помощью инъекций зависимостей (dependency injection, DI), стоит посмотреть на инъекции интерфейсов. Это означает что когда класс реализует определенный интерфейс вы можете внедрить зависимость. Я предполагаю, что у вас уже есть представление о Внедрении Зависимостей (Dependency Injection) и Инверсии Управления (Inversion of Control). Инъекцию интерфейса проще всего объяснить на примере такого кода:

<pre class="lang:php"><code>
interface TweetServiceAwareInterface
{
    public function setTweetService(TweetService $service);
}

class BloggerService implements TweetServiceAwareInterface
{
    public function setTweetService(TweetService $service) {}
}
</code></pre>

И в своей фабрике вы можете проверять наличие интерфейса:

<pre class="lang:php"><code>
class ServiceFactory
{
    public function createService($name)
    {
        // Create service here based on $name

        if ($service instanceof TweetServiceAwareInterface) {
            // Create tweet service
            $service->setTweetService($tweetService);
        }

        return $service;
    }
}
</code></pre>

Как вы могли заметить, здесь есть два важных аспекта:

1. Если вы внедряете сервис твитов проверяя совпадает ли интерфейс с требуемым, то это называется инъекция интерфейса.
2. Сервис это мягкая зависимость: когда сервис не реализует `TweetServiceAwareInterface` сервис твитов не будет устанавливаться, но код продолжит нормально работать.

## Инициализаторы

Теперь вы можете предположить, что инициализаторы в `Zend\ServiceManager` это хороший вариант для использования если вам нужно подобное поведение. Инициализаторы следят за каждым созданным классом и могут "усилить" процесс сборки объекта путем выполнения дополнительной работы, например, внедрения мягких зависимостей.

Давайте вернемся к практическому примеру использования логгера. Вот интерфейс логгера для использования в вашем инициализаторе: `Zend\Log\LoggerAwareInterface`. Предположим, у вас есть сервис, где вы могли бы регистрировать выполнение некоторых действий пользователя. Аналогично описанному выше, вы реализуете интерфейс:

<pre class="lang:php"><code>
namespace MyModule\Service;

use Zend\Log\Logger;
use Zend\Log\LoggerAwareInterface;

class MyService implements LoggerAwareInterface
{
    protected $logger;

    public function setLogger(Logger $logger)
    {
        $this->logger = $logger;
    }

    public function getLogger()
    {
        return $this->logger;
    }

    public function doSomething()
    {
        // Do stuff here

        if (null !== $this->getLogger()) {
            $this->getLogger()->info('User did something here'); 
        }

        // Continue your work
    }
}
</code></pre>

В методе вы можете проверить, есть ли у вас логгер. Если есть, записать сообщение. Если нет, пропустить логирование и продолжить. Заключительная часть здесь - инициализатор, который вам нужно написать. Важно, чтобы вы познакомились с двумя предпосылками для работы инъекции интерфейса:

1. Логгер должен быть сервисом внутри сервис-менеджера. Вы можете создать фабрику для логгера, о чем [я уже рассказывал в недавнем посте][5].
2. Класс `MyService` может быть извлечен из сервис-менеджера. Вы, например, можете указать имя класса сервиса в конфигурации сервис-менеджера.

[5]: /blog/using-zend-framework-service-managers-in-your-application.html

Тогда ваш инициализатор может выглядеть как-то так:

<pre class="lang:php"><code>
namespace MyModule;

use Zend\Log\LoggerAwareInterface;
use Zend\Mvc\MvcEvent;

class Module
{
    public function getServiceConfig()
    {
        return array(
            'initializers' => array(
                'logger' => function($service, $sm) {
                    if ($service instanceof LoggerAwareInterface) {
                        $logger = $sm->get('logger');
                        $service->setLogger($logger);
                    }
                }
            ),
        );
    }
}
</code></pre>

Теперь если вы вызовите `$serviceManager->get('my-service');`, инициализатор увидит, что `MyService` является экземпляром `LoggerAwareInterface`. Он извлечет логгер из сервис-менеджера и внедрит в ваш класс `MyService`.

Это позволяет вам отделить логирование от сервиса и повторно использовать инициализатор для множества классов. Если вы хотите больше классов, использующих логгер, просто добавьте другой класс и сделайте его `LoggerAware`. Если вы хотите заменить экземпляр логгера для всех сервисов, просто замените фабрику логгера. Если вы хотите отключить логирование, удалите инициализатор. Это чрезвычайно гибкий подход и как только вы привыкнете к нему, он станет великолепным инструментом в ваших приложениях.

Несколько последних замечаний:

1. Я сделал инициализатор в виде _замыкания_. Вы так же можете сделать его в виде [callable][6] или класса, реализующего `Zend\ServiceManager\InitializerInterface`;
2. Я включил инициализатор в `getServiceConfig()` класса модуля. Вы можете добавить его в любом месте вашего кода, но не забывайте добавлять его до того, как вы попытаетесь извлечь класс. В противном случае ваш инициализатор не сможет внедрять зависимости. Как [я писал в предыдущем посте][5], ключ `initializers` может быть помещен в конфигурационный файл модуля, внутрь ключа `service_manager`.

[6]: http://php.net/manual/en/language.types.callable.php