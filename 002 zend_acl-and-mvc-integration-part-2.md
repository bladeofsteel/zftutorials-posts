__Источник:__ [Zend_Acl and MVC Integration Part I (Basic Use)][1]  
__Автор:__ Aldemar Bernal  
__Перевод:__ [Лобач Олег][2]

 [1]: http://devzone.zend.com/article/3510-Zend_Acl-and-MVC-Integration-Part-II-Advanced-Use
 [2]: http://lobach.info/

В [первой части](http://devzone.zend.com/article/3509-Zend_Acl-and-MVC-Integration-Part-I-Basic-Use) ([Интеграция Zend_Acl и MVC. Часть 1](/blog/zend_acl-and-mvc-integration-part-i.html)) мы говорили о том, как настроить экземпляр Zend_Acl и включить его в окружение MVC (с использованием плагина фронт-контроллера). Но как же настройки других действий для блокирования доступа, или как сделать редактирование статьи только её автором? Это и кое-что еще будет рассмотрено далее.

Как я уже говорил в первой части, эта статья основана на следующем [предложении](http://framework.zend.com/wiki/pages/viewpage.action?pageId=39025), которое в настоящий момент находится в стадии исследования.

## 1. Использование модулей {#section1}

Давайте поговорим о модулях. В качестве примера для данной статьи мы взяли сайт подобный ДевЗоне. А что если мы создадим административный модуль, задачей которого будет реализация процедуры одобрения статей. Кроме того в модуле можно реализовать еще ряд задач, таких как управление короткими ссылками или категориями. В этом случае нам придется изменить нашу модель ресурсов:

__Общий модуль__

1. Контроллер пользователей.
2. Контроллер статей.

__Административный модуль__

1. Контроллер статей.
2. Контроллер коротких ссылок.
3. Контроллер категорий.

Основываясь на этой новой модели ресурсов, мы создадим экземпляр Zend_Acl, который будет отражает её.

__Примечание__: помните, что этот код и создание экземпляра объекта Zend_Acl должно исполняться перед вызовом метода диспетчеризации фронт-контроллера (во время загрузки).

<pre lang="php"><code>
/** Creating Roles */
require_once 'Zend/Acl/Role.php';
$myAcl->addRole(new Zend_Acl_Role('guest'))
      ->addRole(new Zend_Acl_Role('writer'), 'guest')
      ->addRole(new Zend_Acl_Role('admin'), 'writer');

/** Creating resources */
require_once 'Zend/Acl/Resource.php';
/** Default module */
$myAcl->add(new Zend_Acl_Resource('user'))
      ->add(new Zend_Acl_Resource('article'));

/** Admin module */
$myAcl->add(new Zend_Acl_Resource('admin'))
      ->add(new Zend_Acl_Resource('admin:article'), 'admin')
      ->add(new Zend_Acl_Resource('admin:quick-link'), 'admin')
      ->add(new Zend_Acl_Resource('admin:category'), 'admin');

/** Creating permissions */
$myAcl->allow('guest', 'user')
      ->deny('guest', 'article')
      ->allow('guest', 'article', 'view')
      ->allow(array('writer', 'admin'), 'article', array('add', 'edit'))
      ->allow('admin', 'admin');

/** Setting up the front controller */
require_once 'Zend/Controller/Front.php';
$front = Zend_Controller_Front::getInstance();
$front->setControllerDirectory(array('default' => 'path/to/default/controllers',
                                     'admin' => 'path/to/admin/controllers'));

/** Registering the Plugin object */
require_once 'Zend/Controller/Plugin/Acl.php';
$front->registerPlugin(new Zend_Controller_Plugin_Acl($myAcl, 'guest')); 

/** Dispatching the front controller */
$front->dispatch();
</code></pre>

Отмечу, что как и раньше, мы создали ресурс для каждого контроллера. Но в случае с модулем администратора, мы создали ресурс для модуля и по одному для каждого контроллера внутри этого модуля в формате "модуль:контроллер", сделав их потомками ресурса модуля. Так же мы сделали роль Администратор единственной, кому разрешен доступ ко всему модулю администрирования.

## 2. Использование ролей {#section2}

Как только пользователь вошел в приложение, он должен получить роль. В нашем примере эта роль может быть "гость", "автор" или "администратор". Но как мы можем изменить текущую ACL-роль в нашем компоненте? Во-первых, вы должны сохранить эту роль в переменной, находящейся в пространстве сессии. Таким образом, как только пользователь входит, вы должны сохранить роль пользователя в сессии. При следующем запросе, вы возьмете  эту переменную из сессии и используете её для настройки плагина фронт-контроллера во время фазы загрузки.

__Контроллер пользователей__

<pre lang="php"><code>
class UserController extends Zend_Controller_Action
{
    protected $_application;

    public function init()
    {
        require_once 'Zend/Session/Namespace.php';
        $this->_application = new Zend_Session_Namespace('myApplication');
    }

    public function loginAction()
    {
        ... Validation code
        if ($valid) {
            /** Setting role into session */
            $this->_application->currentRole = $user->role;
            $this->_application->loggedUser = $user->username;
        }
    }

    public function logoutAction()
    {
        $this->_application->currentRole = 'guest';
        $this->_application->loggedUser = null;
    }
}
</code></pre>

__Файл-загрузчик__

<pre lang="php"><code>
/** Loading application from session */
require_once 'Zend/Session/Namespace.php';
$application = new Zend_Session_Namespace('myApplication');

if (!isset($application->;currentRole)) {
    $application->currentRole = 'guest';
}

/** Setting up the front controller */
require_once 'Zend/Controller/Front.php';
$front = Zend_Controller_Front::getInstance();
$front->setControllerDirectory('path/to/controllers'); 

/** Registering the Plugin object */
require_once 'Zend/Controller/Plugin/Acl.php';
$front->registerPlugin(new Zend_Controller_Plugin_Acl($myAcl, $application->currentRole));

/** Dispatching the front controller */
$front->dispatch();
</code></pre>

## 3. Установка действия при ошибке "запрет доступа" {#section3}

Может быть, некоторым из вас (надеюсь, никому =D) просто не нравится идея иметь действие "Доступ запрещен" в контроллере ошибок, или просто хочется назвать его как-либо иначе. Это можно сделать, вызвав метод setErrorPage плагина фронт-контроллера.

<pre lang="php"><code>
/** Setting up the front controller */
require_once 'Zend/Controller/Front.php';
$front = Zend_Controller_Front::getInstance();
$front->setControllerDirectory('path/to/controllers'); 

/** Setting default access denied action */
require_once 'Zend/Controller/Plugin/Acl.php';
$aclPlugin = new Zend_Controller_Plugin_Acl($myAcl, 'guest');
$aclPlugin->setErrorPage('goaway', 'my-error-controller', 'my-module'); 

/** Registering the Plugin object */
$front->registerPlugin($aclPlugin); 

/** Dispatching the front controller */
$front->dispatch();
</code></pre>

Метод setErrorPage может вызываться с указанием только имени действия. В этом случае контроллер и модуль останутся "error" и "default". Так же метод может вызываться с передачей названий действия и контроллера, либо передав все три параметра.

## 4. Использование помощника действий {#section4}

Наконец, мы увидим один из наиболее важных частей этого предложения. До сих пор в нашем примере ДевЗоны мы видели, что мы разрешаем администраторам и авторам редактировать статьи. Но постойте, есть еще недостающая часть нашего приложения. Сейчас если я автор и имею доступ к `article/edit/:id`, это значит, что я имею доступ к редактированию не только своих статей, но и статей остальных авторов! Это не очень хорошо, так ведь? Итак, что мы будем с этим делать? Мы будем управлять этим используя помощник действий, а это значит, что вы сможете получить доступ к нашим ACL внутри любого контроллера, а не только во время загрузки.

Итак, первое, что мы делаем, это регистрация не только нашего плагина фронт-контролера, но и нашего помощника действий в Брокере помощников действий контроллера.

__Файл-загрузчик__

<pre lang="php"><code>
/** Loading application from session */
require_once 'Zend/Session/Namespace.php';
$application = new Zend_Session_Namespace('myApplication');

if (!isset($application->loggedUser)) {
    $application->loggedUser = null;
}

/** Setting up the front controller */
require_once 'Zend/Controller/Front.php';
$front = Zend_Controller_Front::getInstance();
$front->setControllerDirectory('path/to/controllers'); 

/** Registering the Plugin object */
require_once 'Zend/Controller/Plugin/Acl.php';
$front->registerPlugin(new Zend_Controller_Plugin_Acl($myAcl, $application->currentRole)); 

/** Registering the Action Helper object */
require_once 'Zend/Controller/Action/Helper/Acl.php';
require_once 'Zend/Controller/Action/HelperBroker.php';
Zend_Controller_Action_HelperBroker::addHelper(new Zend_Controller_Action_Helper_Acl()); 

/** Dispatching the front controller */
$front->dispatch();
</code></pre>

И после регистрации помощника, мы можем использовать его внутри любого контроллера. Давайте дадим права доступа на изменение только для владельца или любого администратора.

__Контроллер статей__

<pre lang="php"><code>
class ArticleController extends Zend_Controller_Action
{
    protected $_acl;
    protected $_application;

    public function init()
    {
        /** Get our Action Helper */
        $this->_acl = $this->_helper->getHelper('acl'); 

        require_once 'Zend/Session/Namespace.php';
        $this->_application = new Zend_Session_Namespace('myApplication');
    } 

    ...

    public function editAction()
    {
        /** Load article by id */
        $article = new Article($this->_request->;id);

        /** Validate if the user is the owner or an Admin */
        if (($article->author != $this->_application->loggedUser)
             &amp;&amp; ($this->_application->currentRole != 'admin')) {
            $this->_acl->denyAccess();
        } 

        ...
    }
}
</code></pre>

## Заключение {#section5}

Некоторые любители копошиться в мусоре всю свою жизнь пытаются найти недостающее звено (и они так и умрут в поиске его) и некоторые ЗФ-еры состарятся в попытках заставить работать ACL должным образом в их MVC-окружении. Надеюсь, высказанное выше предложение, может оказаться одним из недостающих кусочков в мире ACL + MVC.

В заключении хочу дать рекомендацию. Придерживайтесь принципа "Делай проще": если вам не нужна позарез динамическая загрузка ACL, то его загрузка и настройка вручную - совсем не грех, возможно это лучший способ действий в данной ситуации.

Для более подробного ознакомления с темой вы можете почитать следующее: [Zend_Acl &amp; MVC Integration][3] и небольшой пример реализации подхода, описанного в статье: [Source Code][4]

 [3]: http://framework.zend.com/wiki/pages/viewpageattachments.action?pageId=39025
 [4]: http://framework.zend.com/wiki/download/attachments/39025/ZionFramework.zip
