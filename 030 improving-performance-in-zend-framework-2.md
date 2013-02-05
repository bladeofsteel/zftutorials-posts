title: Повышение производительности в Zend Framework 2
slug: improving-performance-in-zend-framework-2
category: performance
cite: Несколько советов как повысить производительность приложения на Zend Framework 2.
~~~~
__Источник:__ [Improving Performance in Zend Framework 2][1]  
__Автор:__ [Martin Shwalbe][2]  
__Перевод:__ [Лобач Олег][3]  

[1]: http://hounddog.github.com/blog/performance-in-zend-framework-2/
[2]: http://hounddog.github.com/
[3]: http://lobach.info/

Здесь я собираюсь составить список вариантов как улучшить Производительность. Это будет актуальный список, который я  собираюсь обновлять когда найду новый способ увеличения производительности.

## Autoloader Classmap

Как описал Роб Аллен в своем [блоге][4]:

[4]: http://akrabat.com/zend-framework-2/using-zendloaderautoloader/

> Автозагрузчик с помощью карты классов является высокопроизводительным автозагрузчиком. Он использует карту классов, которая, по сути, является простым ассоциативным массивом, где имя каждого класса связано с названием файла на диске, содержащего этот класс.

Это очень простой процесс и занимает всего несколько минут для генерации и включения карты классов для каждого модуля, который вы создали.

### Создание карты классов

> Как вы можете себе представить, создание карты классов вручную должно быстро утомлять. Чтобы этого избежать, Zend Framework 2 содержит PHP-скрипт (classmap_generator.php в каталоге `bin`), который сделает это за вас. Этот инструмент сканирует все каталоги начиная с текущего (либо с указанного при запуске) и создает файл карты классов для каждого класса, который он обнаружит. Пример использования:
> <pre class="lang:sh theme:neon width:720 font:consolas font-size:14 nums:false highlight:0 decode:true"><code>prompt> path/to/zf2/bin/classmap_generator.php -w
Creating class file map for library in '/var/www/project/library'...
Wrote classmap file to '/var/www/project/library/autoload_classmap.php'</code></pre>

## Карта шаблонов

Большинство людей используют "template_path_stack во время разработки. Как указывает [руководство][5], это может вести к снижению производительности.

[5]: http://framework.zend.com/manual/2.0/en/modules/zend.view.quick-start.html#configuration

> Это хорошее решение для быстрой разработки приложения, но потенциально может привести к снижению производительности при промышленной эксплуатации из-за необходимости большого количества статичных вызовов.

После завершения разработки рассмотрите возможность поместить ваши шаблоны в _template_map_ как описано в руководстве.

Я разработал templatemap_generator, который можно найти в моем [gist][6].

[6]: https://gist.github.com/4169214

Запустите этот скрипт в корне вашего модуля, например для `module/Album`:

<pre class="lang:sh theme:neon width:720 font:consolas font-size:14 nums:false highlight:0 decode:true"><code>
$ php ../../vendor/ZF2/bin/templatemap_generator.php
Creating template file map for library in 'zf2-tutorial/module/Album'...
Wrote templatemap file to 'zf2-tutorial/module/Album/template_map.php'
</code></pre>

Теперь вы можете просто подключить этот файл в вашем `module.config.php` следующим образом:

<pre class="lang:php"><code>
<?php
return array(
    'view_manager' => array(
        'template_map' => include __DIR__ .'/../template_map.php',
    ),
);
</code></pre>

Вместо этого, вы так же можете подключить [OcraCachedViewResolver][7] для повышения производительности поиска нужного шаблона путем кэширования.

[7]: https://github.com/Ocramius/OcraCachedViewResolver

## Кеширование конфигурации модуля

Как отметил Bakura, мы так же должны добавить кеширование конфигурации модуля.

Создайте файл `modulecache.local.php` в `config/autoload` следующим образом:

<pre class="lang:php"><code>
<?php
return array(
    // Whether or not to enable a configuration cache.
    // If enabled, the merged configuration will be cached and used in
    // subsequent requests.
    'config_cache_enabled' => true,
    // The key used to create the configuration cache file name.
    'config_cache_key' => 'module_config_cache',
    // The path in which to cache merged configuration.
    'cache_dir' => './data/cache',
);
</code></pre>

Это просто быстрые заметки, которые я продолжу обновлять по мере поступления новых вариантов. Если вы чувствуете, что чего-то не хватает, или следует отметить, пожалуйста, не стесняйтесь комментировать, тогда я смогу обновляет этот список. Спасибо за чтение!
