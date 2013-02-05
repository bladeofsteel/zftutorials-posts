__Источник:__ [Integrating Zend Framework and Doctrine][1]
__Автор:__ Ruben Vermeersch
__Перевод:__ [Лобач Олег](http://lobach.info/)

_Эта статья научит вас всем шагам, необходимым для создания проекта, использующего [Zend Framework][2] и Doctrine. Шаг за шагом мы создадим простейшую доску сообщений._
  
## Прежде, чем начать

Хоть я и старался сохранить статью простой, это не значит, что тут будет введение в обе технологии. Я предлагаю вам поиграть с обеими технологиями по отдельности, прежде чем пытаться объединить их. Они обе имеют достаточно хорошую документацию, что бы с нее начать: [Zend Framework Quick Start][3] и Doctrine's My First Project. Кроме того, [Akra's Zend Framework Tutorial][4] (*"[Введение в Zend Framework][5]" - перевод от Александра Мусаева*) тоже очень хорошее введение в тему.

Zend Framework имеет "используй-как-хочешь" архитектуру. Это значит, что вы свободны в использовании только тех частей, которые вам нужны, в отличии от других фреймворков, предлагающих решения "все-или-ничего". Эта архитектура (используй-как-хочешь) великолепна: она позволяет нам разрабатывать настоящие Zend Framework приложения, без использования ZF абстракции от БД (Zend_Db). Хотя Zend_Db не плохая технология, она все еще довольно низкоуровневая, и слишком близка к базе данных. Используя Doctrine, вы можете манипулировать вашими данными как объектами, не слишком беспокоясь о базе данных.

Zend Framework предоставляет вам много свободы в том, как строить своё приложение. Иными словами, он не заставляет вас использовать строго определенную структуру проекта. В этой статье я старался наиболее близко следовать предлагаемой по умолчанию структура проекта. Тем не менее, все это вопрос личного вкуса.

## Итак, приступим!

Сначала мы создадим типовую структуру проекта и установим библиотеки. Откройте файл-менеджер и создайте структуру папок, как показано ниже. Я объясню, назначение этих папок через минуту.

[caption id="" align="aligncenter" width="211" caption="Типовая структура папок"]<img title="Типовая структура папок" src="/wp-content/uploads/2008/07/basic-folders.png" alt="Типовая структура папок" width="211" height="468" />[/caption]

Тут много папок, но большинство из них должны быть вам знакомы, если вы уже разрабатывали приложения на Zend Framework. Вот отличия:

* `application/doctrine/` — в ней содержаться все файлы данных Doctrine, такие как sql и yaml схемы бд, миграции, дампы, и т.п.
* `application/models/` — Doctrine будет автоматически генерировать файлы моделей в этой директории, что легко использовать в рамках вашего Zend Framework приложения.
* `library/` — как правило, вы бы просто установить свою копию Zend Framework в эту папку. В нашем приложении нам нужны две библиотеки: Zend Framework и Doctrine. Таким образом, мы делаем два подкаталога и устанавливаем туда библиотеки.
* `scripts/` — Doctrine поставляется с удобнымм скриптами командной строки (делее просто "скрипты доктрины"), мы будем хранить их здесь (как в предложении, упомянутом выше).

Следующий шаг — установка Zend Framework и Doctrine. Загрузите последнюю версию с соответствующих веб-сайтов и разархивируйте library (для ZF) или lib (для Doctrine) в созданные нами папки. Должно получиться приблизительно следующее:

[caption id="attachment_61" align="aligncenter" width="430" caption="Zend Framework и Doctrine установлены"]<img class="size-full wp-image-61" title="Zend Framework и Doctrine установлены" src="http://zftutorials.ru/wp-content/uploads/2008/07/libraries-installed.png" alt="" width="430" height="260" />[/caption]

## Время заняться загрузочным файлом

Если вы помните по Zend Framework Quick Start, мы должны создать загрузочный файл. Мы сделаем это сейчас, но с некоторыми изменениями, с тем, чтобы использовать Doctrine.

Во-первых, мы создадим файлы `public/index.php` и `public/.htaccess`. Запустите ваш любимый редактор и скопируйте следующие куски кода:

**public/index.php**

<pre lang="php"><code>
<?php
require '../application/bootstrap.php';
</code></pre>

**public/.htaccess**

<pre lang="apache"><code>
RewriteEngine on

RewriteCond %{SCRIPT_FILENAME} !-f
RewriteRule ^(.*)$ index.php/$1
</code></pre>

Как вы видите, все так же, как и в любом приложении на базе Zend Framework.

Файл `application/bootstrap.php` выглядит немного иначе. Я разделил его на два файла: `application/bootstrap.php` и `application/global.php`. Первый занимается обработкой запросов клиентов, последний подключает все необходимые файлы. Я разделил их потому, что код из `global.php` нужен также в скриптах Doctrine (которое мы увидим через минутку).

**application/global.php**

<pre lang="php"><code>
<?php
error_reporting(E_ALL | E_STRICT);
ini_set('display_startup_errors', 1);
ini_set('display_errors', 1);
date_default_timezone_set('Europe/Brussels');

/*
 * Setup libraries & autoloaders
 */
set_include_path(dirname(__FILE__).'/../library/zendframework'
    . PATH_SEPARATOR . dirname(__FILE__).'/../library/doctrine'
    . PATH_SEPARATOR . dirname(__FILE__).'/models'
    . PATH_SEPARATOR . dirname(__FILE__).'/models/generated'
    . PATH_SEPARATOR . get_include_path());
require 'Zend/Loader.php';
Zend_Loader::registerAutoload('Zend_Loader');

/*
 * Set super-global data
 */
Doctrine_Manager::connection("mysql://user:pass@localhost/database");

/*
 * Configure Doctrine
 */
Zend_Registry::set('doctrine_config', array(
    'data_fixtures_path'  =>  dirname(__FILE__).'/doctrine/data/fixtures',
    'models_path'         =>  dirname(__FILE__).'/models',
    'migrations_path'     =>  dirname(__FILE__).'/doctrine/migrations',
    'sql_path'            =>  dirname(__FILE__).'/doctrine/data/sql',
    'yaml_schema_path'    =>  dirname(__FILE__).'/doctrine/schema'
));
</code></pre>

**application/bootstrap.php**

<pre lang="php"><code>
<?php
require dirname(__FILE__).'/global.php';

Zend_Controller_Front::run(dirname(__FILE__).'/controllers');
?>
</code></pre>

Давайте рассмотрим его шаг за шагом:

<pre lang="php"><code>
<?php
error_reporting(E_ALL | E_STRICT);
ini_set('display_startup_errors', 1);
ini_set('display_errors', 1);
date_default_timezone_set('Europe/Brussels');
?>
</code></pre>

Это всегда хорошая идея — установить правильную обработку ошибок и часовой пояс. Здесь ничего особенного.

<pre lang="php"><code>
<?php
/*
 * Setup libraries & autoloaders
 */
set_include_path(dirname(__FILE__).'/../library/zendframework'
    . PATH_SEPARATOR . dirname(__FILE__).'/../library/doctrine'
    . PATH_SEPARATOR . dirname(__FILE__).'/models'
    . PATH_SEPARATOR . dirname(__FILE__).'/models/generated'
    . PATH_SEPARATOR . get_include_path());
require 'Zend/Loader.php';
Zend_Loader::registerAutoload('Zend_Loader');
?>
</code></pre>

Именно здесь мы подключаем Doctrine. Как вы видите, мы установили include_path для подключения Zend Framework и Doctrine. Мы также подключили папки, в которых Doctrine будет генерировать файлы моделей. Заметим, что нам не нужно настраивать автозагрузчик Doctrine. Использование загрузчика Зенда работает так же хорошо, до тех пор, пока include_path настроен правильно. *Предупреждение: в текущей версии ZF (1.5.1) есть ошибка, приводящая к печати безобидных предупреждений при использовании классов шаблонов Doctrine. Она должна быть исправлена в будущих версиях.*

<pre lang="php"><code>
<?php
/*
 * Set super-global data
 */
Doctrine_Manager::connection("mysql://user:pass@localhost/database");

/*
 * Configure Doctrine
 */
Zend_Registry::set('doctrine_config', array(
    'data_fixtures_path'  =>  dirname(__FILE__).'/doctrine/data/fixtures',
    'models_path'         =>  dirname(__FILE__).'/models',
    'migrations_path'     =>  dirname(__FILE__).'/doctrine/migrations',
    'sql_path'            =>  dirname(__FILE__).'/doctrine/data/sql',
    'yaml_schema_path'    =>  dirname(__FILE__).'/doctrine/schema'
));
?>
</code></pre>

Это последний кусочек кода. Во-первых, мы создали соединение с базой данных. Чтобы не усложнять, я просто забил в код это строковое значение. В реальных системах вы должны использовать что-нибудь подобное [Zend_Config][7]. Я оставлю это вам в качестве домашнего задания. Во-вторых, этот кусок кода настраивает пути к инструментам Doctrine для командной строки (которые генерируют весь код и схемы баз данных). Я сохранил этот массив в [Zend_Registry][8], который является универсальным хранилищем, местом, где вы можете хранить объекты и получить их в нужный момент.

Измените строку соединения Doctrine, в соответствии с параметрами вашей системы. Вы должны указать пустую базу данных. Мы будем наполнять эту базу данных позже.

Содержимое файла `application/bootstrap.php` не должно вызывать у вас удивления. Опять же, я пытался сохранить этот пример как можно более простым.

Наконец, давайте настроим интерфейс командной строки Doctrine:

**scripts/doctrine-cli**

<pre lang="php"><code>
#!/usr/bin/env php
<?php
require dirname(__FILE__).'/../application/global.php';

$cli = new Doctrine_Cli(Zend_Registry::get('doctrine_config'));
$cli->run($_SERVER['argv']);
</code></pre>

Сделайте этот сценарий исполняемым:

<pre lang="sh" class="theme:neon font:consolas font-size:16 nums:false highlight:0 decode:true"><code>chmod +x scripts/doctrine-cli</code></pre>

и вы готовы двигаться дальше.

## Создание приложения

Теперь, когда мы подготовили базовые скрипты, давайте создадим простейшее приложение с использованием Zend Framework и Doctrine. Мы будем создавать очень простую доску сообщений, место, где пользователи могут отправлять сообщения и просматривать чужие сообщения.

Мы будем делать это очень просто: только один контроллер и один скрипт. Скопируйте следующие файлы:

**application/views/scripts/index/index.phtml**

<pre lang="php"><code>
<html>
<head>
    <title>ZF & Doctrine example</title>
</head>

<body>
<h1>Submit a message:</h1>
<?= $this->form?>

<hr />
<h1>Messages posted:</h1>
<!-- TODO: Show messages here -->
</body>
</html>
</code></pre>

**application/controllers/IndexController.php**

<pre lang="php"><code>
<?php
class IndexController extends Zend_Controller_Action
{
    public function indexAction()
    {
        $form = $this->getForm();
        $req = $this->getRequest();
        if ($req->getPost() && $form->isValid($req->getPost())) {
            // TODO: Insert message into database
        }
        $this->view->form = $form;

        // TODO: Retrieve all messages.
    }

    private function getForm()
    {
        $form = new Zend_Form();
        $form->addElement('text', 'name', array(
                'label' => 'Your name',
                'required' => true
        ));
        $form->addElement('textarea', 'message', array(
                'label' => 'Message',
                'required' => true,
                'rows' => 4
        ));
        $form->addElement('submit', 'send');
        return $form;
    }
}
?>
</code></pre>

Как вы видите, остались три больших TODO пункта: один в скрипте и два в контроллере. Это места, где мы будем подключать Doctrine. Но для этого мы должны в первую очередь определить некоторые объекты данных. Мы вернемся к контроллеру и скрипту позже, сейчас пришло время для создания схемы базы данных.

## Определение схемы базы данных

Doctrine позволяет указать ваши схемы баз данных в YAML-файлах, очень простом текстовом представлении. Мы воспользуемся этим и пусть Doctrine генерирует PHP-файлы автоматически. Я определил следующие схему:

**application/doctrine/schema/schema.yml**

<pre lang="yaml"><code>
Message:
    columns:
        id:
            primary: true
            autoincrement: true
            type: integer(4)
        posted:
            type: timestamp
        name:
            type: string(255)
        message:
            type: string
</code></pre>

В этом приложении нам нужен только один простой объект: Сообщение, имеющий 4 поля: обязательный уникальный идентификатор, время, когда сообщение было отправлено, название темы и само сообщение.

Теперь мы можем использовать командную строку Doctrine для создания файлов моделей и таблиц баз данных. Из вашего шелла выполните следующее:

<pre lang="sh" class="theme:neon nums:false font:consolas font-size:16 highlight:0 decode:true"><code>
$ ./scripts/doctrine-cli generate-models-yaml
generate-models-yaml - Generated models successfully from YAML schema
$ ./scripts/doctrine-cli generate-sql
generate-sql - Generated SQL successfully for models
$ ./scripts/doctrine-cli create-tables
create-tables - Created tables successfully
</code></pre>

Если все прошло хорошо, ошибок напечатано не будет. Если это все же произошло, проверьте параметры соединения с БД и указание путей.

## Соеденим все вместе

Теперь давайте окончательно решим куски TODO. Мы заменим их шаг за шагом. Полный код для завершенных файлов имеется в конце статьи. Во-первых, мы добавим код для хранения сообщений. Замените это:

<pre lang="php"><code>
<?php
// TODO: Insert message into database
?>
</code></pre>

Вот этим (игнорируя тэги `<?php` и `?>`):

<pre lang="php"><code>
<?php
$message = new Message();
$message->fromArray($form->getValues(true));
$message->posted = new Doctrine_Expression('NOW()');
$message->save();
?>
</code></pre>

Как вы видите, мы используем объект класса Сообщение. Этот класс был автоматически сгенерирован Doctrine. Вы можете найти его в `application/models/`. Автозагрузчик позаботится о загрузке всего, что потребуется.

Мы также должны получать сообщения, чтобы показать их. Во-первых, контроллер. Заменить:

<pre lang="php"><code>
<?php
// TODO: Retrieve all messages.
?>
</code></pre>

На:

<pre lang="php"><code>
<?php
$messages = Doctrine_Query::create()
        ->from('Message m')
        ->orderBy('m.posted DESC')
        ->execute();
$this->view->messages = $messages;
?>
</code></pre>

Опять же, это очень просто. Я использовал DQL запрос, чтобы сортировать в обратной хронологической последовательности.

Теперь все, что осталось — показать сообщение в нашем скрипте представления. Опять же, замените:

<pre><code class="html">
<!-- TODO: Show messages here -->
</code></pre>

На:

<pre lang="php"><code>
<?php foreach ($this->messages as $message): ?>
    <h2><?=$message->name?> (<?=$message->posted?>)</h2>
    <?=$message->message?>
<?php endforeach; ?>
</code></pre>

И мы сделали, результат должен выглядеть подобно этому:

[caption id="attachment_63" align="aligncenter" width="500" caption="Завершенное приложение"]<img class="size-full wp-image-63" title="Завершенное приложение" src="/wp-content/uploads/2008/07/chatapp-finished.png" alt="Завершенное приложение" width="500" height="636" />[/caption]

Не используйте это приложение в реальной работе — чтобы сосредоточиться на важных частях, я оставил за кадром множество вещей, таких как, например, экранирование ввода. Вы должны добавить это самостоятельно.

## Заключение

Итак, у вас теперь есть чистое и ясное приложение-пример интеграции Zend Framework и Doctrine. Как следование некоторой философии, эта возможность интеграции их очень понятным образом, превращает создание приложений в сплошное удовольствие.

Если у Вас есть какие-либо замечания, комментарии или вопросы, не стесняйтесь, пишите мне по электронной почте (ruben@savanne.be), или оставляйте комментарий в моем [блоге][10].

## Приложение: Полный код файлов

**application/controllers/IndexController.php**

<pre lang="php"><code>
<?php
class IndexController extends Zend_Controller_Action
{
    public function indexAction()
    {
        $form = $this->getForm();
        $req = $this->getRequest();
        if ($req->getPost() && $form->isValid($req->getPost())) {
            $message = new Message();
            $message->fromArray($form->getValues(true));
            $message->posted = new Doctrine_Expression('NOW()');
            $message->save();
        }
        $this->view->form = $form;

        $messages = Doctrine_Query::create()
            ->from('Message m')
            ->orderBy('m.posted DESC')
            ->execute();
        $this->view->messages = $messages;
    }

    private function getForm()
    {
        $form = new Zend_Form();
        $form->addElement('text', 'name', array(
                'label' => 'Your name',
                'required' => true
        ));
        $form->addElement('textarea', 'message', array(
                'label' => 'Message',
                'required' => true,
                'rows' => 4
        ));
        $form->addElement('submit', 'send');
        return $form;
    }
}
?>
</code></pre>

**application/views/scripts/index/index.phtml**

<pre lang="php"><code>
<html>
<head>
    <title>ZF & Doctrine example</title>
</head>

<body>
    <h1>Submit a message:</h1>
    <?=$this->form?>

    <hr />
    <h1>Messages posted:</h1>
    <?php foreach ($this->messages as $message): ?>
        <h2><?=$message->name?> (<?=$message->posted?>)</h2>
        <?=$message->message?>
    <?php endforeach; ?>
</body>
</html>
</code></pre>

 [1]: http://ruben.savanne.be/articles/integrating-zend-framework-and-doctrine
 [2]: http://framework.zend.com/
 [3]: http://framework.zend.com/docs/quickstart
 [4]: http://akrabat.com/zend-framework-tutorial/
 [5]: http://archive.paradigm.ru/zend-fw-intro.pdf
 [6]: /wp-content/uploads/2008/07/libraries-installed.png
 [7]: http://framework.zend.com/manual/en/zend.config.html
 [8]: http://framework.zend.com/manual/en/zend.registry.html
 [10]: http://weblog.savanne.be/130-gsoc-zf-doctrine