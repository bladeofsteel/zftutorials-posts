Предлагаю вашему вниманию мой первый опыт на ниве видео-презентаций и скринкастов.
Крайне хочется услышать отзывы, посему не почтите за труд, выскажитесь в комментариях к этому посту или в фейсбуке/гугл+

<iframe width="770" height="433" src="https://www.youtube.com/embed/i1Mu4sD9UZI" frameborder="0" allowfullscreen></iframe>

## Команды, использованные в видео

* Загрузка проекта

    <pre lang="sh" class="lang:sh theme:neon width:700 font:consolas font-size:16 nums:false highlight:0 decode:true"><code>wget -O - https://github.com/zendframework/ZendSkeletonApplication/tarball/master | tar xz --strip-components=1</code></pre>

* Обновление composer
 
    <pre lang="sh" class="lang:sh theme:neon width:700 font:consolas font-size:16 nums:false highlight:0 decode:true"><code>php composer.phar self-update</code></pre>

* Установка зависимостей
 
    <pre lang="sh" class="lang:sh theme:neon width:700 font:consolas font-size:16 nums:false highlight:0 decode:true"><code>php composer.phar install</code></pre>

*  конфиг виртуального хоста

    <pre class="width:700 lang:apache decode:true "><code>
    <VirtualHost *:80>
        ServerName      blog.local

        DocumentRoot "/home/oleg/projects/blog/public"
        <Directory "/home/oleg/projects/blog/public/">
            Options Indexes FollowSymLinks MultiViews
            AllowOverride All
            Order allow,deny
            Allow from all
        </Directory>
    </VirtualHost></code></pre>
