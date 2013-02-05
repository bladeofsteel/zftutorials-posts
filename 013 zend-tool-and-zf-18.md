__Источник:__ [Zend_Tool and ZF 1.8](http://devzone.zend.com/article/4559-Zend_Tool-and-ZF-1.8)
__Автор:__ Ralph Schindler 
__Переводчик:__ [Лобач Олег](http://lobach.info/)

Вы уже наверняка слышали новость о выпуске Zend Framework 1.8. Вместе с ZF 1.8 в наше распоряжение поступило несколько новых инструментов, таких как Zend_Tool (это мой любимчик), Zend_Application, Zend_Navigation и Zend_Tag_Cloud. Zend_Tool не является компонентом в обычном понимании. Большинство компонентов имеют класс на верхнем уровне пространства имен, а Zend_Tool нет. Большинство компонентов обычно используется внутри кода вашего приложения для упрощения выполнения задач, а Zend_Tool нет. Zend_Tool более близок к фреймворку, нежели к компоненту - этакий фреймворк в фреймворке.

## Так что же такое Zend_Tool? {#section1}

Первый шаг - приступить к изучению того, что нужно предпринять для разработки следующего поколения фреймворка для RAD[^1]. RAD, как вы можете себе представить, это термин, имеющий довольно широкое определение. В общем смысле, этот термин обозначает скорость, с которой вы можете создать ресурсы, требуемые вашему приложению. В идеальной ситуации, начальная разработка (или подготовительная фаза проекта[^2]) должна быть сведена к минимуму с тем, чтобы разработчики могли приступить к более интересной разработке. В конце концов, "интересная разработка" является наиболее вероятной причиной рождения приложения.

[^1]: Rapid Application Development - [быстрая разработка приложений](http://ru.wikipedia.org/wiki/RAD)
[^2]: loading phase of a project

Итак, если есть "интересная разработка", что же такое "неинтересная разработка"? Для Zend Framework (и большинства MVC-фреймворков, если уж на то пошло) это процесс создания начальных ресурсов проекта, общих для всех проектов: первичная структура проекта, первичные файлы конфигурации, первичный загрузочный код и код автозагрузки, и т.п. Это так же включает задачи обработки ошибок, создания контроллеров и представлений и многое другое. Для кого-то, кто только начинает увлекаться ZF, некоторые из этих задач потребуют часы чтения учебников, руководств и материалов типа "быстрый старт". Не слишком "быстро", не так ли?

## Zend_Tool как фреймворк {#section2}

Вместо тесного связывания системы, которая была бы специально нацелена на создание ZF-приложений непосредственно из командной строки, и точной генерации кода, без модификации существующего кода, мы заложили в архитектуру системы расширяемость в местах, которые должны касаться разработчика. Zend_Tool был разработан для облегчения абстракции во всех необходимых местах, которые мы посчитали удобными разработчикам для расширения системы. Например, у нас пока основной клиент - CLI[^3], но фреймворк разработан как универсальная RPC-система, с тем, чтобы разрабатывать другие, не-CLI, клиенты. Хотя у нас уже есть запас провайдеров (базовые возможности системы), интерфейсы для построения провайдеров в будущем остается простым и удобным для расширения и понимания.

[^3]: Command Line Interface - интерфейс командной строки

## Zend_CodeGenerator и Zend_Reflection {#section3}

В ходе разработки, мы обнаружили несколько проблемных областей, которые не имели единодушного решения. Генерация кода - одна из таких областей. Вообще говоря, когда Вы говорите о генерации кода, это обычно подход, основанный на шаблонах - генерируемый код поступает из файлов шаблонов, которые обычно написаны без учета правильно ли они сформированы[^4] (в соответствии со стандартами языка). Имея подход, основанный на шаблонах, включает риск того, что разработчики выйдут за рамки стандартов кодирования и будут генерировать вообще плохой код (плохой код в этом смысле - это и неработающий код, и плохо сформированный код). Итак, мы сразу поняли, что существует возможность создать компонент, который будет обеспечивать объектно-ориентированный интерфейс, аналогичный [Reflection API](http://ru.php.net/manual/ru/language.oop5.reflection.php) в PHP, с единственной целью - создание хорошо сформированного и хорошо отформатированного объектно-ориентированного кода. Следует отметить, что компонент Zend_CodeGenerator может быть использован без остальной части Zend_Tool. Это означает, что если вы когда-нибудь окажетесь в положении, когда нужно постоянно генерировать код, в первую очередь вы должны посмотреть в сторону Zend_CodeGenerator.

[^4]: well-formed - синтаксически корректны

Zend_CodeGenerator не просто пишет код с нуля; у него есть возможность чтения существующего кода, изменения его, и создания нового кода. Это главным образом связано с другим компонентом - Zend_Reflection, рожденным вне Zend_Tool. Zend_Reflection - это не, и я это особо подчеркну, переизобретение колеса. Фактически, он расширяет Reflection API, добавляя поддержку пользовательских расширений, отражений Dockbloc (и в том числе тэгов dockbloc), и основанных на файлах отражений. Семантика компонента такая же, как у прародителя, и он может использоваться в качестве замены базовых классов в случае необходимости в дополнительных возможностях.

Zend_CodeGenerator и Zend_Reflection имеют схожие API: Zend_Reflection нацелен на чтение структур кода, а Zend_CodeGenerator нацелен на написание структур кода. Вместе эти два компонента облегчают исследование и написание кода во время процесса развития приложения.

## Клиент командной строки Zend_Tool {#section4}

Само собой разумеется, что RAD, основанный на интерфейсе командной строки, очень востребован у ZF-разработчиков. Как упомянуто ранее, задача настраивания начальных ресурсов проекта может быть утомительной. Многие разработчики предпочитают взаимодействовать со средой разработки через терминал, или командную строку, так что, естественно, мы решили, что наиболее целесообразно встроить клиента в Zend_Tool. Это не означает, что может быть только один клиент - как упомянуто ранее, клиентские функциональные возможности Zend_Tool были абстрагированы таким образом, что дополнительные клиенты могут быть построены так, чтобы взаимодействовать с Zend_Tool. У интегрированных сред разработки и редакторов текста есть возможность встроиться в клиентский фреймворк и подавать команды посредством их родного интерфейса. Два вероятных расширения включают два моих любимых инструмента: Zend Studio и Textmate - но возможности почти безграничны. Любой клиент, который способен выполнять PHP-код, может эффективно надстроиться  поверх Zend_Tool для нужд инструментов.

## Zend_Tool_Project {#section5}

В связи с тем, что разработка проекта это итерационный процесс, со стороны инструментария так же необходимо отслеживать историю, если можно так выразиться. То, что мы подразумеваем под историей, отслеживает действия, которые вы выполнили: что вы создали, где это расположено в структуре проекта, и каков контекст этого в проекте? Например, после создания проекта, вы можете захотеть создать контроллер. Для всех намерений и целей, контроллер это просто файл, с одним классом в нем, как может инструментальная среда узнать разницу между обычным файлом и файлом контроллера?

Zend_Tool_Project намеревается решать эту проблему. Zend_Tool_Project это набор функциональности выстроенной для работы с Zend_Tool_Framework, чтобы предоставить решение проблемы управления проектами. Zend_Tool_Project отслеживает ресурсы внутри вашего приложения, где они находятся по отношению друг к другу, и названия, которыми вы обращаетесь к ним, что является ключевым моментом, который дает возможность "итерационную разработку". Например, если вы имеете созданный проект с контроллером, названным "Foo", вы можете позже захотеть добавить действие в этот контроллер. Чтобы сделать разработку настолько гладкой, на сколько возможно, изменение должно быть таким же простым, как создание новых ресурсов. Чтобы сделать это, Zend_Tool_Project отслеживает все, что вы уже сделали в вашем проекте.

Помимо отслеживания ресурсов приложения, Zend_Tool_Project является ключевой частью Zend_Tool, которая обеспечивает решение "построение проекта, основанного на Zend Framework". Zend_Tool_Project точно знает, что такое проект, контроллер, представление, загрузочный класс, файл index.php, и т.д., как должны выглядеть, и помогает вам в их создании. Если вы извлечете Zend_Tool_Project из среды выполнения Zend_Tool, у вас останется только фреймворк (или платформа) для построения инструментальной системы. Это говорит о том, что любой проект может использовать Zend_Tool_Framework для создания инструментов, обеспечивающих их нужды.

## На что оно способно сейчас {#section6}

Итак, имеем в виду все вышесказанное. Что оно может сделать прямо сейчас? Вместо того чтобы говорить об этом, давайте посмотрим несколько скриншотов.

### Помощь

<a href="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool1.png"><img src="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool1-300x270.png" alt="Zend_Tool. Помощь" title="Zend_Tool. Помощь" width="300" height="270" class="aligncenter size-medium wp-image-151" /></a>

### Ошибка

<a href="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool2.png"><img src="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool2-300x275.png" alt="Zend_Tool. Ошибка" title="Zend_Tool. Ошибка" width="300" height="275" class="aligncenter size-medium wp-image-152" /></a>

### Создание проекта

<a href="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool3.png"><img src="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool3-300x235.png" alt="Zend_Tool. Создание проекта" title="Zend_Tool. Создание проекта" width="300" height="235" class="aligncenter size-medium wp-image-153" /></a>

### Создание контроллера

<a href="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool4.png"><img src="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool4-300x235.png" alt="Zend_Tool. Создание контроллера" title="Zend_Tool. Создание контроллера" width="300" height="235" class="aligncenter size-medium wp-image-154" /></a>

### Создание действия

<a href="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool5.png"><img src="http://zftutorials.ru/wp-content/uploads/2009/05/zf_tool5-300x235.png" alt="Zend_Tool. Создание действия" title="Zend_Tool. Создание действия" width="300" height="235" class="aligncenter size-medium wp-image-155" /></a>

## Что дальше

В дополнение к тому, что уже сделано в релизе 1.8, ряд дополнительных возможностей уже в процессе разработки. Некоторые из этих возможностей уже в статусе бета-версии (и это причина, почему они не в дистрибутиве релиза), или все еще в фазе проектирования. К примеру, Zend_Application - мы приняли решение о том, как "модель" должна выглядеть, пусть даже всего лишь мы говорим о названии. Так же, с релизом 1.8 мы опубликовали структуру проекта по-умолчанию. Для проекта, основой которого является "библиотека компонентов", это огромный шаг. 

В дополнение к поддержке моделей, мы планируем добавить поддержку "моделей" (для построения компонентных приложений), соединений с базой данных и генерацию файлов Zend_Db_Table. Ищите эти возможности в ближайших релизах. Кроме того, поскольку Zend_Tool так расширяем, новые возможности могут быть вынесены за пределы проектов, внедряемых непосредственно в модуль Zend_Tool.