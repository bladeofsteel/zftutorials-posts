title: Как добавить элемент формы с квадратными скобками?
slug: how-add-form-element-named-with-brackets
cite: Периодически возникает задача получить в форме элементы, которые в получаемых от клиента данных представляются в виде массива.
~~~~
## Введение

Периодически возникает задача получить в форме элементы, которые в получаемых от клиента данных представляются в виде массива. В html этои элементы могут выглядеть так:

<pre class="lang:html"><code>
<input type="text" name="element[1]" id="element-1" value="">
<input type="text" name="element[2]" id="element-2" value="">
</code></pre>

При получении данных от формы в них будет параметр `element` содержащий массив с ключами 1 и 2, а значениями будут данные, введенные пользоателем в эти поля.

В руководстве нет описания как сделать это с помощью `Zend_Form`. Но сложного ничего нет. Есть минимум 3 способа.

## Простой массив значений

Если нужно из формы получить просто массив значений и ключи не важны, можно воспользоваться методом `setIsArray`, чтобы указать, что этот элемент входит в состав массива:

<pre class="lang:php"><code>
protected function addElement()
{
    $element = new Zend_Form_Element_Text("element");
    $element->setLabel('element 1')->setIsArray(true);
    $this->addElement($element, 'e1');

    $element = new Zend_Form_Element_Text("element");
    $element->setLabel('element 2')->setIsArray(true);
    $this->addElement($element, 'e2');
}
</code></pre>

Обратите внимание на добавление элемента в форму - вторым параметром указывается уникальный идентификатор, иначе добавляемые элементы будут затирать предыдущие.

## Указание на предыдущий элемент

Если вам нужно самостоятельно указывать индекс элемента в результирующем массиве, то можно воспользоваться методом `setBelongsTo` для указания общего названия для всех элементов. Название конкретного элемента, которое нужно указывать при его создании, будет использоваться в качестве индекса. Следует отметить, что название индекса должно иметь строковой тип. Если вам нужны числовые ключи, их надо приводить к типу string.

<pre class="lang:php"><code>
protected function addElement()
{
    $element = new Zend_Form_Element_Text("foo");
    $element->setLabel('element 1')->setBelongsTo('element');
    $this->addElement($element);

    $element = new Zend_Form_Element_Text("bar");
    $element->setLabel('element 2')->setBelongsTo('element');
    $this->addElement($element);
}
</code></pre>

## Использование подчиненных форм

Как и в предыдущем примере, этот способ можно использовать для указания конкретных ключей получаемого массива. В этом случае название элемента будет использовано в качестве ключа, а название формы - в качестве общего названия элементов.

<pre class="lang:php"><code>
protected function addElement()
{
    $subForm = new Zend_Form_SubForm();

    $element = new Zend_Form_Element_Text("1");
    $element->setLabel('element 1');
    $subForm->addElement($element);

    $element = new Zend_Form_Element_Text("2");
    $element->setLabel('element 2');
    $subForm->addElement($element);

    $this->addSubForm($subForm, 'element');
}
</code></pre>