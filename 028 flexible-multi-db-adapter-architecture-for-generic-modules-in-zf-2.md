title: Гибкая архитектура с несколькими адаптерами к базам данных для общих модулей в ZF 2
slug: flexible-multi-db-adapter-architecture-for-generic-modules-in-zf-2
category: database
cite: Очередная статья о применении сервис-менеджера. На этот раз показано использование абстрактных фабрик для реализации гибкой архитектуры, позволяющей легко подменить используемый модулем адаптер к базе данных.
<blockquote>…Основной целью при разработке основного модуля было обеспечить возможность быстрого переключения между различными адаптерами к базам данных (Zend\Db, Doctrine ORM, Doctrine Mongo…) без необходимости что-либо переписывать.</blockquote>
~~~~
__Источник:__ [Flexible multi db-adapter architecture for generic modules in ZF 2][1]  
__Автор:__ [Michaël Gallego][2]  
__Перевод:__ [Лобач Олег][3]  

[1]: http://www.michaelgallego.fr/blog/2012/10/23/flexible-multi-db-adapter-architecture-for-generic-modules-in-zf-2/
[2]: http://www.michaelgallego.fr/blog/
[3]: http://lobach.info/

В настоящий момент я разрабатываю [модуль Форум][4] для Zend Framework 2. Основной целью при разработке основного модуля было обеспечить возможность быстрого переключения между различными адаптерами к базам данных (Zend\Db, Doctrine ORM, Doctrine Mongo…) без необходимости что-либо переписывать.

[4]: https://github.com/zf-fr/ZfrForum

В настоящее время модуль базируется только на Doctrine ORM, но мне хотелось бы поддерживать Zend\Db так же хорошо, как и Doctrine Mongo, следоватьельно ZfrForum должен быть модулем "по умолчанию" с Zend\Db, ZfrForumDoctrineORM - модулем с поддержкой DoctrineORM, а ZfrForumDoctrineODM - модулем с поддержкой Doctrine Mongo.

В этой статье я покажу вам как я решил эту проблему и как абстрактные фабрики (одна из наименее известных возможностей ZF2, я полагаю!) могут радикально уменьшить количество строк, которые вам придется написать.

## Принцип

Основная идея довольно проста: мы собираемся создать интерфейсы мапперов и конкретные реализации под каждый адаптер, который мы хотим поддерживать (Zend\Db, Doctrine ORM, Doctrine ODM…).

Сервис будет пользоваться этими мапперами используя их интерфесы (но не конкретную реализацию!). Таким образом, сервис-менеджер будет внедрять либо маппер Zend\Db, либо Doctrine ORM репозиторий.

## Интерфейсы

Во-первых, мы создадим интерфейс, который представляет собой один маппер. Для тех, кто не знает, "маппер" - это просто объект, который взаимодействует с базой данных и возвращает объекты. Например, вот простой маппер для работы с категориями:

<pre class="lang:php"><code>
<?php
namespace ZfrForum\Mapper;

use ZfrForum\Entity\Category;

interface CategoryMapperInterface
{
    /**
     * @param  Category $category
     * @return mixed
     */
    public function create(Category $category);

    /**
     * @param  Category $category
     * @return mixed
     */
    public function update(Category $category);
}
</code></pre>

## Конкретная реализация

Теперь, когда у нас есть интерфейс, напишем конкретные классы.

По умолчанию, это маппер, использующий внутри Zend\Db (я сам никогда не использовал Zend\Db, так что код может быть слегка не аккуратным, прошу прощения за это!).

<pre class="lang:php"><code>
namespace ZfrForum\Repository;

use Zend\Db\TableGateway\TableGatewayInterface;
use ZfrForum\Entity\Category;
use ZfrForum\Mapper\CategoryMapperInterface;

class CategoryMapper implements CategoryMapperInterface
{
    /**
     * @var TableGatewayInterface
     */
    protected $tableGateway;

    public function __construct(TableGatewayInterface $tableGateway)
    {
        $this->tableGateway = $tableGateway;
    }

    /**
     * @param  Category $category
     * @return mixed
     */
    public function create(Category $category)
    {
        // TODO: Implement create() method.
    }

    /**
     * @param  Category $category
     * @return mixed
     */
    public function update(Category $category)
    {
        // TODO: Implement update() method.
    }
}
</code></pre>

С другой стороны, ZfrForumDoctrineORM может иметь что-то, что будет выглядеть так (помните, что мапер в этом случае наследуется от EntityRepository, так что часть операций уже реализована за нас, таких как find, findBy…). Но так как мы реализуем наш CategoryMapperInterface, оно продолжает работать!

<pre class="lang:php"><code>
namespace ZfrForum\Repository;

use Doctrine\ORM\EntityRepository;
use ZfrForum\Entity\Category;
use ZfrForum\Mapper\CategoryMapperInterface;

class CategoryRepository extends EntityRepository implements CategoryMapperInterface
{
    /**
     * @param  Category $category
     * @return mixed
     */
    public function create(Category $category)
    {
        // TODO: Implement create() method.
    }

    /**
     * @param  Category $category
     * @return mixed
     */
    public function update(Category $category)
    {
        // TODO: Implement update() method.
    }
}
</code></pre>

## Сервисы

Теперь давайте напишем сервисный слой. Как вы можете себе представить, сервисный слой имеет зависимость от маппера (потому что сервисный слой не должен напрямую взаимодействовать с базой данных, это задача маппера). Конечно, идея не в том, чтобы использовать ZfrForum\Mapper\CategoryMapper или ZfrForum\Repository\CategoryRepository, НО в использовании интерфейса (ZfrForum\Mapper\CategoryMapperInterface) так, что реализация сервиса может переключаться между различными реализациями маппера.

Вот CategoryService:

<pre class="lang:php"><code>
namespace ZfrForum\Service;

use ZfrForum\Entity\Category;
use ZfrForum\Mapper\CategoryMapperInterface;

class CategoryService
{
    /**
     * @var CategoryMapperInterface
     */
    protected $categoryMapper;

    /**
     * @param CategoryMapperInterface $categoryMapper
     */
    public function __construct(CategoryMapperInterface $categoryMapper)
    {
        $this->categoryMapper = $categoryMapper;
    }
}
</code></pre>

## Управление зависимостями

CategoryService требуется маппер, реализующий CategoryMapperInterface. Что ж, давайте добавим эти строки в наш файл Module.php что бы обработать эту зависимость (замечу, что я так же позаботился о том, что ZfrForum\Mapper\CategoryMapper нуждается в зависимости в виде объекта TableGateway).

<pre class="lang:php"><code>
public function getServiceConfig()
{
    return array(
        'factories' => array(
            'ZfrForum\Service\CategoryService' => function($serviceManager) {
                $categoryMapper = $serviceManager->get('ZfrForum\Mapper\CategoryMapperInterface');
                return new Service\CategoryService($categoryMapper);
            },

            'ZfrForum\Mapper\CategoryMapper' => function($serviceManager) {
                 $dbAdapter = $sm->get('Zend\Db\Adapter\Adapter');
                 $resultSetPrototype = new ResultSet();
                 $resultSetPrototype->setArrayObjectPrototype(new Category());
                 return new TableGateway('category', $dbAdapter, null, $resultSetPrototype);
            },
        ),
        /**
         * Abstract factories
         */
        'abstract_factories' => array(
            'ZfrForum\ServiceFactory\MapperAbstractFactory'
        )
    );
}
</code></pre>

Конечно, возникла новая проблема: сервис-менеджер не знает что такое "ZfrForum\Mapper\CategoryMapperInterface". Мы могли бы, конечно, добавить следующий псевдоним в нашу конфигурацию сервиса:

<pre class="lang:php"><code>
// В ZfrForum :
return array(
    'aliases' => array(
        'ZfrForum\Mapper\CategoryMapperInterface' => 'ZfrForum\Mapper\CategoryMapper'
    )
);

// В ZfrForumDoctrine :
return array(
    'aliases' => array(
        'ZfrForum\Mapper\CategoryMapperInterface' => 'ZfrForum\Repository\CategoryRepository'
    )
);
</code></pre>

Но представьте, что у вас 15 мапперов! Вам нужно 15 псевдонимов! И, конечно, так как другой модуль может использовать другие реализации (ZfrForum использует ZfrForum\Mapper\CategoryMapper, а ZfrForumDoctrineORM - ZfrForum\Repository\CategoryRepository), вы должны дублировать эти 15 строк в обоих модулях!

Конечно, есть способ лучше: абстрактные фабрики. Обратите внимание в предыдущем куске кода на ключ "abstract_factories".

Действительно, когда вы вызываете функцию сервис-менеджера "get", он будет выполнять следующие шаги:

1. Во-первых, он проверит находится ли имя в списке имен классов. Если да, то он напрямую создает объект, иначе переходит к шагу 2.
2. Он проверяет связано ли имя с одной из фабрик. Если да, он использует фабрику для создания объекта, иначе переходит к шагу 3.
3. Он переберает каждую абстрактную фабрику и спрашивает её может ли она создать объект с таким именем. Если абстрактная фабрика ответит "да", она будет использована для создания объекта.

Вот этот механизм мы и будем использовать. Это может работать, потому что мы считаем, что все наши интерфейсы мапперов имеют следующую структуру: "ZfrForum\Mapper\xxxxMapperInterface", где "xxxx" - имя сущности, которая мапится.

Таким образом, наша абстрактная фабрика для Zend\Db выглядит примерно так:

<pre class="lang:php"><code>
namespace ZfrForum\ServiceFactory;

use Zend\ServiceManager\AbstractFactoryInterface;
use Zend\ServiceManager\ServiceLocatorInterface;

class MapperAbstractFactory implements AbstractFactoryInterface
{
    /**
     * Determine if we can create a service with name
     *
     * @param ServiceLocatorInterface $serviceLocator
     * @param                         $name
     * @param                         $requestedName
     * @return bool
     */
    public function canCreateServiceWithName(ServiceLocatorInterface $serviceLocator, $name, $requestedName)
    {
        return (substr($requestedName, -15) === 'MapperInterface');
    }

    /**
     * Create service with name
     *
     * @param ServiceLocatorInterface $serviceLocator
     * @param                         $name
     * @param                         $requestedName
     * @return mixed
     */
    public function createServiceWithName(ServiceLocatorInterface $serviceLocator, $name, $requestedName)
    {
        // Mapper names given are under the form "ZfrForum\Mapper\SomethingMapperInterface", so we need to
        // remove the MapperInterface part
        $parts       = explode('\\', $requestedName);
        $entityName  = substr(end($parts), 0, -15);
        $entityClass = 'ZfrForum\\Mapper\\' . $entityName . 'Mapper';

        return $serviceLocator->get($entityClass);
    }
}
</code></pre>

Как вы можете видеть, функция "canCreateServiceWithName" проверяет заканчивается ли имя на "MapperInterface". Если да, мы можем что-нибудь сделать и возвратить  true. С другой стороны, функция "createServiceWithName" выполняет некоторые операции со строками, чтобы извлечь имя сущности и создать маппер.

Для Doctrine, это даже проще, так как маппер сущностей может построить для нас репозиторий. Конечно, чтобы это работало, вы должны сказать Doctrine, что репозиторий для категорий это "ZfrForum\Repository\CategoryRepository" (с помощью аннотаций, например).

<pre class="lang:php"><code>
<?php
namespace ZfrForum\ServiceFactory;

use Zend\ServiceManager\AbstractFactoryInterface;
use Zend\ServiceManager\ServiceLocatorInterface;

class MapperAbstractFactory implements AbstractFactoryInterface
{
    /**
     * Determine if we can create a service with name
     *
     * @param ServiceLocatorInterface $serviceLocator
     * @param                         $name
     * @param                         $requestedName
     * @return bool
     */
    public function canCreateServiceWithName(ServiceLocatorInterface $serviceLocator, $name, $requestedName)
    {
        return (substr($requestedName, -15) === 'MapperInterface');
    }

    /**
     * Create service with name
     *
     * @param ServiceLocatorInterface $serviceLocator
     * @param                         $name
     * @param                         $requestedName
     * @return mixed
     */
    public function createServiceWithName(ServiceLocatorInterface $serviceLocator, $name, $requestedName)
    {
        /** @var \Doctrine\ORM\EntityManager $entityManager */
        $entityManager = $serviceLocator->get('Doctrine\ORM\EntityManager');
        
        // Mapper names given are under the form "ZfrForum\Mapper\SomethingMapperInterface", so we need to
        // remove the MapperInterface part
        $parts       = explode('\\', $requestedName);
        $entityName  = substr(end($parts), 0, -15);
        $entityClass = 'ZfrForum\\Entity\\' . $entityName;

        return $entityManager->getRepository($entityClass);
    }
}
</code></pre>

## Заключение

Работа с интерфейсами позволяет проще переключаться между адаптерами, в то время как абстрактная фабрика позволяет нам легко создавать объект, имеющий некое сходство (в данном случае, у всех объектов имя заканчивается на "MapperInterface").

Для примера, если мы захотим поддерживать Doctrine Mongo, это будет так же легко, как создать новый модуль ZfrForumDoctrineMongo, написать конкретную реализацию для каждого маппера, и переопределить абстрактные фабрики, чтобы они возвращали объект документного маппера вместо маппера, основанного на Zend\Db.