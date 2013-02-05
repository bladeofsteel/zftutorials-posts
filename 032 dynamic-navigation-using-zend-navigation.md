title: Динамическая навигация с использованием Zend\Navigation
slug: dynamic-navigation-using-zend-navigation
category: 
cite: Небольшая заметка о создании динамических меню с помощью Zend\Navigation с пошаговым алгоритмом и примерами кода.
<blockquote>Но иногда нам нужно динамическое меню, генерируемое в нашем приложении. Я дам вам простой пример как это можно сделать.</blockquote>
~~~~
__Источник:__ [Zend Framework 2 : Dynamic Navigation using Zend\Navigation][1]  
__Автор:__ [Abdul Malik Ikhsan][2]  
__Перевод:__ [Лобач Олег][3]  

[1]: http://samsonasik.wordpress.com/2012/11/18/zend-framework-2-dynamic-navigation-using-zend-navigation/
[2]: http://samsonasik.wordpress.com/
[3]: http://lobach.info/

Zend Framework 2 для генерации меню предоставляет компонент Navigation. Его можно использовать, создав структуру меню в конфигурационном файле в `autoload/*.global.php`. Но иногда нам нужно динамическое меню, генерируемое в нашем приложении. Я дам вам простой пример как это можно сделать.

Например, я хочу постоить меню подобное этому:

![Скриншот меню](http://zftutorials.ru/wp-content/uploads/2012/12/menu.png)

Таким образом, мы должны делать шаг за шагом, следующее:

1 Создадим таблицу menu (я использую postgresql)

<pre class="lang:sql"><code>
CREATE TABLE menu
(
  id bigserial NOT NULL,
  name character varying(255),
  label character varying(255),
  route character varying(255),
  CONSTRAINT pk_menu PRIMARY KEY (id )
)
WITH (
  OIDS=FALSE
);
ALTER TABLE menu
  OWNER TO postgres;
</code></pre>

2 Вставим данные в таблицу, например такие:

![Данные в таблице](http://zftutorials.ru/wp-content/uploads/2012/12/data-menu.png)

3 Создадим модель:

<pre class="lang:php"><code>
namespace ZendSkeletonModule\Model;

use Zend\Db\Adapter\Adapter;
use Zend\Db\ResultSet\HydratingResultSet;
use Zend\Db\TableGateway\AbstractTableGateway;
use Zend\Db\Sql\Select;
use Zend\Db\Adapter\AdapterAwareInterface;

class MenuTable extends AbstractTableGateway
    implements AdapterAwareInterface
{
    protected $table = 'menu';

    public function setDbAdapter(Adapter $adapter)
    {
        $this->adapter = $adapter;
        $this->resultSetPrototype = new HydratingResultSet();

        $this->initialize();
    }

    public function fetchAll()
    {
        $resultSet = $this->select(function (Select $select){
                $select->order(array('id asc'));
        });

        $resultSet = $resultSet->toArray();

        return $resultSet;
    }
}
</code></pre>

4 Рсширим `Zend\Navigation\Service\DefaultNavigationFactory` и переопределим метод `getPages()`.

<pre class="lang:php"><code>
namespace ZendSkeletonModule\Navigation;

use Zend\ServiceManager\ServiceLocatorInterface;
use Zend\Navigation\Service\DefaultNavigationFactory;

class MyNavigation extends DefaultNavigationFactory
{
    protected function getPages(ServiceLocatorInterface $serviceLocator)
    {
        if (null === $this->pages) {
            //FETCH data from table menu :
            $fetchMenu = $serviceLocator->get('menu')->fetchAll();

            $configuration['navigation'][$this->getName()] = array();
            foreach($fetchMenu as $key=>$row)
            {
                $configuration['navigation'][$this->getName()][$row['name']] = array(
                    'label' => $row['label'],
                    'route' => $row['route'],
                );
            }
            
            if (!isset($configuration['navigation'])) {
                throw new Exception\InvalidArgumentException('Could not find navigation configuration key');
            }
            if (!isset($configuration['navigation'][$this->getName()])) {
                throw new Exception\InvalidArgumentException(sprintf(
                    'Failed to find a navigation container by the name "%s"',
                    $this->getName()
                ));
            }

            $application = $serviceLocator->get('Application');
            $routeMatch  = $application->getMvcEvent()->getRouteMatch();
            $router      = $application->getMvcEvent()->getRouter();
            $pages       = $this->getPagesFromConfig($configuration['navigation'][$this->getName()]);

            $this->pages = $this->injectComponents($pages, $routeMatch, $router);
        }
        return $this->pages;
    }
}
</code></pre>

5 Создадим собственный Navigation Factory

<pre class="lang:php"><code>
namespace ZendSkeletonModule\Navigation;

use Zend\ServiceManager\FactoryInterface;
use Zend\ServiceManager\ServiceLocatorInterface;

class MyNavigationFactory implements FactoryInterface
{
    public function createService(ServiceLocatorInterface $serviceLocator)
    {
        $navigation =  new MyNavigation();
        return $navigation->createService($serviceLocator);
    }
}
</code></pre>

6 Регистрируем модель MenuTable и свой Navigation в сервис-менеджере:

<pre class="lang:php"><code>
class Module
{
    public function getServiceConfig()
    {
        return array(
            'initializers' => array(
                function ($instance, $sm) {
                    if ($instance instanceof \Zend\Db\Adapter\AdapterAwareInterface) {
                        $instance->setDbAdapter($sm->get('Zend\Db\Adapter\Adapter'));
                    }
                }
            ),

            'factories' => array(
                'menu' => function($sm){
                    $menutable = new   Model\MenuTable;
                    return $menutable;
                },

                'Navigation' => 'ZendSkeletonModule\Navigation\MyNavigationFactory'
        ));
    }
}
</code></pre>

7 Настройка завершена, давайте вызовем из макета:

  <pre class="lang:php"><code>
<div class="nav-collapse">
<?php echo $this->navigation('Navigation')->menu()->setUlClass('nav'); ?>
</div>
  </code></pre>

Готово!