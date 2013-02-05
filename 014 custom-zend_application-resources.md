__Источник:__ [Custom Zend_Application Resources](http://akrabat.com/zend-framework/custom-zend_application-resources/)
__Автор:__ [Rob Allen](http://akrabat.com/)
__Переводчик:__ [Лобач Олег](http://lobach.info/)

Рано или поздно, вы захотите использовать Zend_Application эффективней при помощи создания собственных плагинов ресурсов. Это значительно упрощает и ускоряет начальную стадию разработки нового приложения за счет повторного использования однажды уже проделанной работы по инициализации окружения. К тому же это сделает ваш Boostrap-класс очень компактным!

В моем случае, я хотел создать ресурс для [CouchDb](http://couchdb.apache.org/), который проверял бы что база данных была создана, в противном случае создавал бы её.

Создание собственных плагинов достаточно просто. Очевидным местом для размещения будет `library/App/Application/Resource` и типичный ресурс будет выглядеть следующим образом:


<pre><code class="php">class App_Application_Resource_Couchdb 
                extends Zend_Application_Resource_ResourceAbstract
{
    /**
     * Defined by Zend_Application_Resource_Resource
     *
     * @return Phly_Couch|null
     */
    public function init()
    {
        // тут выполняются действия для инициализации CouchDb
        $options = $this->getOptions(); 
        // в $options находится все содержимое 'resources.couchdb' из application.ini
    }
}</code></pre>

Вы должны уведомить Zend_Application о ваших новых плагинах. Это делается посредством одной строчки в application.ini:

<pre class="nums:false lang:default highlight:0 decode:true "><code>pluginPaths.App_Application_Resource_ = "App/Application/Resource"</code></pre>

Теперь вы можете иметь столько плагинов ресурсов, сколько вам захочется, располагая их в пространстве App_Application_Resource_.

Кроме того, Matthew Weier O'Phinney написал статью о [Zend_Application](http://www.weierophinney.net/matthew/archives/230-Quick-Start-to-Zend_Application_Bootstrap.html) (перевод: "[Быстрый старт с Zend_Application_Bootstrap][1]"), которую вам обязательно стоит прочесть.

[1]: /blog/quick-start-to-zend_application_bootstrap.html