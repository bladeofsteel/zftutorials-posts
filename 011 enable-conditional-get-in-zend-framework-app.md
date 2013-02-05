__Источник:__ [Enable your Zend Framework App with Conditional GET!](http://smartycode.com/performance/zend-framework-browser-caching/)  
__Автор:__ Danila Vershinin  
__Перевод:__ [Лобач Олег](http://lobach.info/)

В этой статье я покажу вам простой подход, позволяющий вашим приложениям на Zend Framework снизить нагрузку на каналы связи, став таким образом более дружелюбными к пользователю.

Эта техника предполагает использование условного GET-запроса (HTTP conditional GET). Это базовая возможность HTTP-протокола. Посылая правильные HTTP-заголовки, ваше приложение позволяет браузерам посетителей кэшировать страницы вашего сайта.

Вы беспокоитесь о посетителях, имеющих старые версии страниц в кэше? Не стоит! Предлагаемый метод позволяет получить все выгоды от кэширования на стороне клиента без внесения каких-либо изменений, и требует всего 5 минут вашего времени для ее интеграции :).

Zend Framework великолепен в том, что вы можете легко расширить его. Мы собираемся создать плагин фронт-контроллера, который будет заботиться о обработке условных GET-запросов.

Давайте создадим наш плагин фронт-контроллера:

<pre lang="php"><code>
<?php
/**
 * Plugin to support conditional GET for php pages (using ETag)
 * Should be loaded the very last in the plugins stack
 * 
 * @author $Author: danila $
 * @version $Id: Conditional.php 15741 2009-02-08 11:58:44Z danila $
 *
 */ 
class Smartycode_Http_Conditional extends Zend_Controller_Plugin_Abstract
{

    public function dispatchLoopShutdown()
    {
        $send_body = true;
        
        $etag = '"' . md5($this->getResponse()->getBody()) . '"';
        
        $inm = split(',', getenv("HTTP_IF_NONE_MATCH"));
        
        $inm = str_replace('-gzip', '', $inm);
        
        // TODO If the request would, without the If-None-Match header field, 
        // result in anything other than a 2xx or 304 status, 
        // then the If-None-Match header MUST be ignored
        
        foreach ($inm as $i) {
            if (trim($i) == $etag) {
                $this->getResponse()
                     ->clearAllHeaders()
                     ->setHttpResponseCode(304)
                     ->clearBody();
                $send_body = false;
                break;
            }
        }
        
        $this->getResponse()
             ->setHeader('Cache-Control', 'max-age=7200, must-revalidate', true)
             ->setHeader('Expires', gmdate('D, d M Y H:i:s', time() + 2 * 3600) . ' GMT', true)
             ->clearRawHeaders();
        
        if ($send_body) {
            $this->getResponse()
                 ->setHeader('Content-Length', strlen($this->getResponse()->getBody()));
        } 
        
        $this->getResponse()->setHeader('ETag', $etag, true);
        $this->getResponse()->setHeader('Pragma', '');
        
    }
}
</code></pre>

Подключить этот плагин к фронт-контроллеру очень легко. Так же легко, как добавление строки в загрузочный файл:

<pre lang="php"><code>
$frontController->registerPlugin(
    new Smartycode_Http_Conditional(), 
    101
);
</code></pre>

Обратите внимание на "101". Вы должны зарегистрировать плагин последним в стеке плагинов.

Эти простые шаги сделают ваше приложение на Zend Framework более дружелюбным к окружению:

- Работа AJAX-запросов происходит через зендовский MVC (все виды запросов)
- Если страницы не изменялись со времени последнего запроса, то они не будут передаваться
- Можно также полагать, что вы получите пользу для SEO - поисковые системы, поддерживающие Etag, смогут эффективно пропускать загрузку / повторный анализ страниц сайта, что ускорит индексацию страниц вашего сайта
- Отправка заголовка Content-Length включает постоянные соединения (Keep-Alive connections)
- Есть еще достоинства, но мне лень о них думать