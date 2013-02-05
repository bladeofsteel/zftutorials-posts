__Источник:__ [File uploads with Zend_Form_Element_File][2]
__Автор:__ [Rob Allen][1]
__Перевод:__ [Лобач Олег](http://lobach.info/)

 [1]: http://akrabat.com/
 [2]: http://akrabat.com/zend-framework/file-uploads-with-zend_form_element_file/

Теперь, когда выпущен Zend Framework 1.7, я решил, что стоит взглянуть на встроенный элемент загрузки файлов - `Zend_Form_Element_File`, и посмотреть как можно его использовать. Предлагаю пример на базе простейшей формы.

Я решил использовать тот же набор элементов формы, что и [ранее][3] с целью демонстрации упрощения разработки.

 [3]: http://akrabat.com/zend-framework/simple-zend_form-file-upload-example/

![Пример формы](/wp-content/uploads/2008/12/zend-form-element-file-example.png)

Итак, приступим:

## Форма

Мы расширим `Zend_Form` и сохраним полученный класс `forms_UploadForm` в каталоге `application/forms`:

<pre><code class="php"><?php

class forms_UploadForm extends Zend_Form
{
    public function __construct($options = null)
    {
        parent::__construct($options);
        $this->setName('upload');
        $this->setAttrib('enctype', 'multipart/form-data');

        $description = new Zend_Form_Element_Text('description');
        $description->setLabel('Description')
                  ->setRequired(true)
                  ->addValidator('NotEmpty');

        $file = new Zend_Form_Element_File('file');
        $file->setLabel('File')
            ->setDestination(BASE_PATH . '/data/uploads')
            ->setRequired(true);

        $submit = new Zend_Form_Element_Submit('submit');
        $submit->setLabel('Upload');

        $this->addElements(array($description, $file, $submit));

    }
}</code></pre>

Как и ранее (*прим. пер.: в предыдущей заметке, ссылка на которую была выше*), мы задаем имя формы и атрибут enctype для включения механизма загрузки файлов. Сама форма имеет два поля: текстовое поле, названное "Описание", и поле для загрузки файла, названное "Файл", наряду с кнопкой "Отправить". Здесь ничего особо сложного нет.

Элемент `Zend_Form_Element_File` имеет метод `setDestination()`, который используется для указания нижележащему (*прим. пер.: имеется в виду "ниже в иерархии классов"*) `Zend_File_Transfer_Adapter_Http` где мы хотим сохранить загруженные файлы. В данном примере мы выбрали каталог `data/uploads`.

## Контроллер и представление

Контроллер так же весьма стандартный:

<pre><code class="php"><?php

class IndexController extends Zend_Controller_Action
{
    public function indexAction()
    {
        $this->view->headTitle('Home');
        $this->view->title = 'Zend_Form_Element_File Example';
        $this->view->bodyCopy = "<p>Please fill out this form.</p>";

        $form = new forms_UploadForm();

        if ($this->_request->isPost()) {
            $formData = $this->_request->getPost();
            if ($form->isValid($formData)) {

                // success - do something with the uploaded file
                $uploadedData = $form->getValues();
                $fullFilePath = $form->file->getFileName();

                Zend_Debug::dump($uploadedData, '$uploadedData');
                Zend_Debug::dump($fullFilePath, '$fullFilePath');

                echo "done";
                exit;

            } else {
                $form->populate($formData);
            }
        }

        $this->view->form = $form;

    }
}</code></pre>

Представление `views/scripts/index.phtml` абсолютно тривиально:

<pre><code class="php"><h1><?= $this->title; ?></h1>
<?= $this->bodyCopy; ?>
<?= $this->form; ?></code></pre>

Если форма успешно прошла валидацию, массив `$uploadedData` будет содержать значения полей формы вместе с именем загруженного файла. Обратите внимание, что имя файла не полное. Если вам нужен весь путь к файлу, то используйте метод `getFileName()` файлового элемента.

### Заключение

Это все, что нужно для построения простейшей формы загрузки файлов. Однако есть еще несколько настолько важных ошибок в компоненте, что имеет смысл дождаться версии 1.7.2. К примеру, валидатор `Count` не всегда работает так, как от него ожидаешь, а `getValues()` и `receive()` все еще не достаточно умны, чтобы вызывать `move_uploaded_file()` только один раз.

Как обычно, вот zip-файл проекта, созданный мной для тестирования: [Zend_Form_Element_File_Example.zip][4] (включает Zend Framework (снепшот ветки release-1.7) - именно поэтому он более 3,9Мб).

 [4]: http://akrabat.com/wp-content/uploads/2008/Zend_Form_Element_File_Example.zip