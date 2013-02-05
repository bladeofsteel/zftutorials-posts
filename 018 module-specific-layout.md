Иногда возникает необходимость для конкретного модуля установить собственный макет. Например, обычно макет административной области отличается от макета публичной области сайта. Чтобы этого добиться есть очень простой способ.

[Evan Coury][1] написал небольшой модуль (меньше 20 строк кода), который позволяет указать в конфиге используемый в модуле макет. Модуль опубликован на [github-е][2] и доступен в репозитории [ EvanDotPro / EdpModuleLayouts][3].

## Установка модуля

Для установки можно воспользоваться Composer-ом. Для этого нужно добавить в секцию `require` конфигурационного файла composer-а строку `"evandotpro/edp-module-layouts": "1.*"`. Для пустого проекта секция будет выглядеть следующим образом:

<pre lang="js"><code>
    "require": {
        "php": ">=5.3.3",
        "zendframework/zendframework": "2.*",
        "evandotpro/edp-module-layouts": "1.*"
    }
</code></pre>

Далее нужно обновить проект с помощью composer-а:

<pre lang="sh" class="theme:neon font:consolas font-size:16 nums:false highlight:0 decode:true"><code>php composer.phar update</code></pre>

Результат комманды будет выглядеть примерно так:

![Результат работы composer][4]

Вместо composer можно использовать механизм submodule у git, либо скачать и распаковать тарбол репозитория. Но установка через composer мне видится наиболее простым способом.

После завершения работы composer, нужно подключить установленный модуль к приложению. Для этого в `config/application.config.php` добавляем название модуля:

<pre lang="php"><code>
<?php
return array(
    'modules' => array(
        'EdpModuleLayouts', // <= сюда добавили новый модуль
        'Application',
    ),
    'module_listener_options' => array(
        'config_glob_paths'    => array(
            'config/autoload/{,*.}{global,local}.php',
        ),
        'module_paths' => array(
            './module',
            './vendor',
        ),
    ),
);
</code></pre>

Все, теперь модуль установлен и доступен для использования.

## Подключение индивидуального макета

Для изменения используемого модулем макета нужно добавить в конфигурационный файл модуля либо в файл конфигурации автолоадинга следующий код:

<pre lang="php"><code>
    array(
        'module_layouts' => array(
            'ModuleName' => 'layout/some-layout',
        ),
    );
</code></pre>

Естественно, нужно исправить название модуля и название макета. Например так:

<pre lang="php"><code>
<?php
// config/autoload/global.php
return array(
     'module_layouts' => array(
        'Application' => 'layout/application',
        ),
);
</code></pre>

Я создал новый макет в модуле `Applacation` и в конфигурации указал, что модуль `Application` должен его использовать.

Следует обратить внимание на то, что (по крайней мере пока) названия макетов должны быть уникальными. Хорошим решением будет называть макеты по имени модуля, в котором они используются.

На этом все. Успехов в разрботке!

*PS: если вам помогла эта заметка, то лучший способ отблагодарить меня - нажать на одну из кнопок соц.сетей прямо под этим текстом.*

[1]: http://evan.pro/
[2]: https://github.com/
[3]: https://github.com/EvanDotPro/EdpModuleLayouts
[4]: /wp-content/uploads/2012/10/diffrent-module-layouts_update-project.png