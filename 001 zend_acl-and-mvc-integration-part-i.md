__Источник:__ [Zend_Acl and MVC Integration Part I (Basic Use)][1]  
__Автор:__ Aldemar Bernal  
__Перевод:__ [Лобач Олег][2]

 [1]: http://devzone.zend.com/article/3509-Zend_Acl-and-MVC-Integration-Part-I-Basic-Use
 [2]: http://lobach.info/

Итак, что не так с Zend_Acl и текущей реализацией MVC в Zend Framework? Ничего неправильного нет, просто не слишком очевидно для разработчиков, как достичь оптимальной интеграции между этими двумя важными частями фреймворка.

Во-первых, эта статья основана на следующем предложении ([link][3]), в настоящий момент находящемся в стадии *Ожидания рекомендации*.

 [3]: http://framework.zend.com/wiki/pages/viewpage.action?pageId=39025

Ну, как это работает? Существуют два основных компонента в этом предложении:

1. Плагин фронт-контроллера (Front Controller Plugin): этот компонент решает, имеет ли доступ текущий пользователь к открываемой странице.  
2. Помощник действия (Action Helper): Этот компонент позволяет проверить, имеет ли текущий пользователь доступ внутрь контроллера.

Опираясь на эти два компонента, давайте попробуем их на примере. Давайте будем говорить о сайте, подобном DevZone. Нам потребуется контроллер для управления пользователями и еще один контроллер для управления статьями, так же 3 типа пользователей (ролей): одну для гостей, одну для авторов статей и еще одну для утверждения статей. Итого, мы имеем:

**Ресурсы:**

1. Контроллер пользователей.  
2. Контроллер статей.

**Роли:**

1. Гость (Guest).  
2. Автор (Writer).  
3. Администратор (Admin).

## Настройка компонента Zend_Acl {#section1}

После определения того, что нам нужно сделать, следующим шагом будет создание экземпляра Zend_Acl, отражающего нашу модель.

[php]
/** Creating the ACL object */
require_once 'Zend/Acl.php';
$myAcl = new Zend_Acl();
[/php]

## Создание ролей {#section2}

Сейчас мы создадим роли в нашем экземпляре Zend_Acl.

[php]
/** Creating Roles */
require_once 'Zend/Acl/Role.php';
$myAcl->addRole(new Zend_Acl_Role('guest'))
      ->addRole(new Zend_Acl_Role('writer'), 'guest')
      ->addRole(new Zend_Acl_Role('admin'), 'writer');
[/php]

## Создание ресурсов {#section3}

Создадим необходимые ресурсы (по одному на контроллер), а также их отношения с созданными нами ролями.

[php]
/** Creating resources */
require_once 'Zend/Acl/Resource.php';
$myAcl->add(new Zend_Acl_Resource('user'))
      ->add(new Zend_Acl_Resource('article'));
[/php]

## Создание привилегий {#section4}

Теперь мы добавили роли и ресурсы в наш экземпляр Zend_Acl, пора объяснить, какие действия должны быть доступны для каких ролей.

1. Гости не могут редактировать, добавлять и публиковать статьи.  
2. Авторы не могут публиковать статьи.  
3. Администраторы имеют полный доступ.

[php]
/** Creating permissions */
$myAcl->allow('guest', 'user')
      ->deny('guest', 'article')
      ->allow('guest', 'article', 'view')
      ->allow('writer', 'article', array('add', 'edit'))
      ->allow('admin', 'article', 'approve');
[/php]

## Создание страницы, отображаемой при отсутствии доступа {#section5}

Нам нужно будет создать представление (view) и действие (action) на которое мы переадресуем всех пользователей, у которых недостаточно привилегий.  
Во-первых, мы создадим новое действие в нашем контроллере ошибок:

[php]
class ErrorController extends Zend_Controller_Action
{
    ....

    public function deniedAction()
    {
    }

    ....
}
[/php]

Затем мы создадим наш файл представления (`/application/views/scripts/error/denied.phtml`) с некоторым предупреждающим сообщением:

[html]
<h1>Error</h1>
<h2>Access denied</h2>
<p>You are trying to access an area which you have not allowed.</p>
[/html]

### Завершение настройки {#section16}

Хорошо, мы настроили наш экземпляр Zend_Acl. Следующий шаг - регистрация плагина контроллера. Это важная часть берет созданный нами экземпляр Zend_Acl и проверяет, доступна ли текущая страница пользователю.

<pre lang="php"><code>
/** Setting up the front controller */
require_once 'Zend/Controller/Front.php';
$front = Zend_Controller_Front::getInstance();
$front->setControllerDirectory('path/to/controllers'); 

/** Registering the Plugin object */
require_once 'Zend/Controller/Plugin/Acl.php';
$aclPlugin = new Zend_Controller_Plugin_Acl($myAcl);
$aclPlugin->setRoleName($currentUserRole);

$front->registerPlugin(new Zend_Controller_Plugin_Acl($acl, 'guest')); 

/** Dispatching the front controller */
$front->dispatch();
</code></pre>

После завершения настройки, как только пользователь войдет в наше приложение, в зависимости от его/её роли будет либо отображена запрошенная страница, либо страница с сообщением о запрете доступа.

Для более подробного ознакомления с темой вы можете почитать следующее:  
[Zend_Acl & MVC Integration](http://framework.zend.com/wiki/pages/viewpageattachments.action?pageId=39025)

и небольшой пример: [Source Code](http://framework.zend.com/wiki/download/attachments/39025/ZionFramework.zip)