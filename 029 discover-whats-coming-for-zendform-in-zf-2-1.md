title: Что нового в Zend\Form появится с версией ZF 2.1
slug: discover-whats-coming-for-zendform-in-zf-2-1
category: form
cite: Первый взгляд на новые возможности, которые появятся в Zend\Form с выходом ZF 2.1. Новые элементы для выбора даты, упрощение управлением зависимостями, использование сервис-менеджера и много другое ждет нас в новой версии фреймворка, которая должна появиться со дня на день.
<blockquote>Следуя политике Zend Framework 2, никаких нарушений совместимости с предыдущими версиями в 2.1 не ожидается, так что вы можете … безопасно перейти на 2.1 и просто наслаждаться новыми возможностями.</blockquote>
~~~~
__Источник:__ [Discover what’s coming for Zend\Form in ZF 2.1][1]  
__Автор:__ [Michaël Gallego][2]  
__Перевод:__ [Лобач Олег][3]  

[1]: http://www.michaelgallego.fr/blog/2012/11/09/discover-whats-coming-for-zendform-in-zf-2-1/
[2]: http://www.michaelgallego.fr/blog/
[3]: http://lobach.info/

Как вам возможно известно, ожидается что самый первый минорный выпуск Zend Framework 2 (ZF 2.1) должены показать в конце месяца. Следуя политике Zend Framework 2, никаких нарушений совместимости с предыдущими версиями в 2.1 не ожидается, так что вы можете (окей… вы могли бы, я не хочу никаких неприятностей :) ) безопасно перейти на 2.1 и просто наслаждаться новыми возможностями.

Я собираюсь поговорить здесь о главных новых возможностях Zend\Form, надеюсь вам понравится!

## Новые элементы

ZF 2.1 приходит с двумя новыми элементами, которые написал я: DateSelect и MonthSelect. В отличии от обычных элементов Date или Month (которые отображаются как текстовое поле или с элементом для выбора даты в некоторых браузерах, таких как Опера), DateSelect и MonthSelect отображаются используя три выпадающих списка (или два для MonthSelect). В некоторых случаях (день рождения…) это, безусловно, для пользователя удобнее, чем выбор даты с помощью календаря.

Создать такой элемента крайне просто:

<pre class="lang:php"><code>
$this->add(array(
    'type'    => 'Zend\Form\Element\DateSelect',
    'name'    => 'birthDate',
    'options' => array(
        'label' => 'Your birthdate',
        'create_empty_option' => true,
    )
));
</code></pre>

Эти элементы принимают много новых параметров для тонкой настройки:

<pre class="lang:php"><code>
$this->add(array(
    'type'    => 'Zend\Form\Element\DateSelect',
    'name'    => 'birthDate',
    'options' => array(
        'label'               => 'Birthdate',
        'create_empty_option' => true,
        'min_year'            => date('Y') - 30,
        'max_year'            => date('Y') - 18,
        'day_attributes'      => array(
              'style'            => 'width: 22%'
        ),
        'month_attributes'    => array(
             'style'            => 'width: 35%'
        ),
        'year_attributes'     => array(
            'style'            => 'width: 25%'
        )
    )
));
</code></pre>

Как вы видите, этот элемен принимает различные параметры:

- create_empty_option: если установлен в _true_, будут автомитически созданы варианты, чьи значения пусты для всех трех списков (это полезно для многих JavaScript-библиотек, которые ожидают от вас пустые варианты для их последующего наполнения).
- min_year и max_year: минимальный и максимальный год для… элемента выбора года (капитан очевидность).
- day_attributes, month_attributes, year_attributes: они позволяют установить атрибуты, которые будут применены к спискам.

Конечно, MonthSelect работает так же, за исключением того, что у него нет элемента выбора дня.

Вы можете поинтересоваться валидацией и локализацией? Не стоит! Прелесть в том, что они автомитически провалидируют дату за вас. Что касается локализации, они используют данные Intl, что означает, что порядок "день-месяц-год", как и разделитель ("день-месяц-год", "день/месяц/год") и даже название месяцев автоматически локализуются используя текущую локаль.

Обратите внимание на то, что если вы вызовите getValue у такого элемента, он вернет дату в следеющем формате: 'Y-m-d'. Это позволяет вам конвертировать её в объекте DateTime в любой нужный формат.

## FormElementManager

Это следующая большая вещь. Form теперь следует за остальными компонентами и принимает PluginHelperManager, который позволяет… ну, много вещей.

## Короткие имена

Прежде всего, вам больше не надо писать полный название элемента. Это означает что это:

<pre class="lang:php"><code>
$this->add(
    'type' => 'Zend\Form\Element\Url',
    'name' => 'url'
);
</code></pre>

можно записать как:

<pre class="lang:php"><code>
$this->add(
    'type' => 'Url',
    'name' => 'url'
);
</code></pre>

Это работает потому, что теперь элементы вытаскиваются из сервис-менеджера. Это означает, что вы можете также переопределить стандартный элеменит Url и использовать свой собственный.

## Зависимости

Вытаскивание объектов из сервис-менеджера так же означает, что управлять зависимостями становится гораздо проще. 

Предположим, что вашему Fieldset нужен ServiceManager. Раньше вам приходилось переносить его из Form
 в Fieldset (и если вы пользуетесь, как я, очень глубокой структурой, вам может понадобится перемещать его мнного-много раз… хотя он нужен только последнему в иерархии).

Теперь просто сделайте Fieldset реализующим ServiceLocatorAwareInterface:

<pre class="lang:php"><code>
use Zend\Form\Fieldset;
use Zend\ServiceManager\ServiceLocatorAwareInterface;
use Zend\ServiceManager\ServiceLocatorInterface;

class MyFieldset extends Fieldset implements ServiceLocatorAwareInterface
{
    protected $serviceLocator;

    public function setServiceLocator(ServiceLocatorInterface $serviceLocator)
    {
        $this->serviceLocator = $serviceLocator;
    }

    public function getServiceLocator()
    {
        return $this->serviceLocator;
    }
}
</code></pre>

И добавьте fieldset в вашу форму:

<pre class="lang:php"><code>
use Zend\Form\Form;

class MyForm extends Form
{
    public function init()
    {
        $this->add(array(
            'type' => 'MyFieldset',
            'name' => 'fieldset'
        ));
    }
}
</code></pre>

Теперь, вместо прямой инициализации fieldset, используйте имя, это приведет к тому, что в первую очередь будет предпринята попытка получить объект из сервис-менеджера. Так же вы можете определить зависимости используя новый метод getFormElementConfig в вашем классе Module.php:

<pre class="lang:php"><code>
public function getFormElementConfig()
{
    return array(
        'factories' => array(
            'MyFieldset' => function($sm) {
                // Your code
            }
        )
    );
}
</code></pre>

Конечно, чтобы это заработало, вы должны инициализировать вашу форму также используя сервис-менеджер. То есть, в вашем контроллере, вместо этого:

<pre class="lang:php"><code>
public function testAction()
{
    $form = new MyForm();
}
</code></pre>

писать так:

<pre class="lang:php"><code>
public function testAction()
{
    $form = $this->serviceLocator()->get('FormElementManager')->get('MyForm');
}
</code></pre>

Мы сначала получаем класс FormElementManager (который по сути является классом PluginManager), а затем получаем форму. Обратите внимание, вам не нужно объявлять "MyForm" в секции "invokables", так как неизвестные строки по умолчанию будут рассматриваться как элементы этого типа.

Кроме того, так как большинство зависимостей устанавливаются через инициализаторы (например, ServiceLocatorAwareInterface), это означает, что добавление элементов в конструкторе будет преждевременным, так как зависимости через инициализаторы еще не установлены. Именно поэтому мы добавили метод “init”, который вызывается после того, как все зависимости будут установлены.

Так что в вашем fieldset, вместо добавления элементов в __construct, просто добавьте их в “init” (да, как в стиле старого ZF1!).

## Полностью новая загрузка файлов

Последнее по порядку, но не последнее по значению. Спасибо [cgmartin][4], загрузчик файлов был полностью обновлен. Это то, что мы не успели обновить к ZF 2.0, следовательно, обновленный файл был в полной неразберихе и совершенно не работал с новой архитектурой форм.

[4]: https://github.com/cgmartin

Теперь это полностью исправлено и Крис для этого проделал поразительную работу! Я предлагаю вам взглянуть на примеры: [https://github.com/cgmartin/ZF2FileUploadExamples][5]

[5]: https://github.com/cgmartin/ZF2FileUploadExamples

Я надеюсь, формы ZF 2 вам понравяться еще больше! Наслаждайтесь!