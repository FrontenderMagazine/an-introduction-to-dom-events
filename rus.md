# Введение в события DOM

Список событий, возможных в DOM, очень длинный: `click` (клик мышью), `touch` (касание), `load` (загрузка), `drag` (перетягивание), `change` (изменение), `input` (ввод), `error` (ошибка), `resize` (изменение размера) и т.д. События могут срабатывать для любой части документа вследствие взаимодействия с ним пользователя или браузера. Они не просто начинаются и заканчиваются в одном месте; они циркулируют по всему документу, проходя свой собственный жизненный цикл. Это и делает события DOM столь гибкими и полезными. Разработчики должны понимать как работают события DOM, чтобы суметь использовать их потенциал и построить увлекательный интерфейс. 

За всё время работы фронтенд-разработчиком у меня сложилось впечатление, что ни в одном из просмотренных мной источников я не видел чёткого объяснения как работают события DOM. Моя цель в этой статье предоставить полный обзор этой темы, чтобы помочь вам освоить её быстрее, чем это удалось мне. 

Я опишу основы работы с событиями DOM, затем углублюсь во внутренние аспекты работы и объясню как их можно использовать для решения распространённых практических задач. 

## Обработка событий

В прошлом процесс присоединения обработчиков событий к узлам DOM в браузерах был довольно непоследовательным. Библиотеки вроде [jQuery][1] играли бесценную роль в абстрагировании от связанных с ним странностей. 

Чем ближе мы подходим к стандартизированным браузерным окружениям, тем более безопасным становится использование API из официальной спецификации. Чтобы не усложнять вещи, я опишу как можно управлять событиями именно в современном вебе. Если вы пишите JavaScript для Internet Explorer (IE) 8 или старше, рекомендую для управления обработкой событий использовать [полифил][2] или фреймворк (такой как [jQuery][3]).

В JavaScript обработчик события можно установить используя следующее:

    element.addEventListener(<event-name>, <callback>, <use-capture>);

* `event-name` (строка). Это имя или тип события, которое вы хотите обработать. Им может быть любое стандартное событие DOM (`click`, `mousedown`, `touchstart`, `transitionEnd` и т.д.) или даже имя вашего пользовательское события (о пользовательских событиях мы поговорим позже).
* `callback` (функция). Эта функция вызывается когда происходит событие. Объект `event`, содержащий информацию о событии, присваивается в качестве первого параметра.
* `use-capture` (булево значение). Оно объявляет должен ли обработчик события вызываться в фазе «перехвата». (Не беспокойтесь, я объясню что это значит немного позже)

    var element = document.getElementById('element');

    function callback() {
      alert('Привет');
    }

    // Добавление обработчика события
    element.addEventListener('click', callback);

[Демо: `addEventListener`][4]

## Удаление приемника события

Удаление приемника события, когда он больше не нужен, считается хорошим тоном (особенно в веб-приложениях с длительным временем выполнения). Чтобы это сделать, используйте метод `element.removeEventListener()`:

    element.removeEventListener(<event-name>, <callback>, <use-capture>);

У `removeEventListener`, однако, есть один нюанс: нужно указать функцию обратного вызова, которая изначально была привязана к событию. Вызов просто `element.removeEventListener('click');` работать не будет. 

По существу, если мы хотим иметь возможность удалять приемники событий (это необходимо делать для «долгоиграющих» приложений), нам нужно следить за нашими функциями обратного вызова. Это значит, что нельзя использовать анонимные функции. 

    var element = document.getElementById('element');

    function callback() {
      alert('Привет один раз');
      element.removeEventListener('click', callback);
    }

    // Добавление приемника события
    element.addEventListener('click', callback);

[Демо: removeEventListener][5]

### Поддержание контекста обработчика события в рабочем состоянии

Сбой в программе очень просто получить благодаря вызову обработчика события с некорректным контекстом. Объясним это на примере.

    var element = document.getElementById('element');

    var user = {
     firstname: 'Виктор',
     greeting: function(){
       alert('Меня зовут ' + this.firstname);
     }
    };

    // Присоединяем user.greeting в качестве обработчика события
    element.addEventListener('click', user.greeting);

    // alert => 'Меня зовут undefined'

[Демо: Некорректный контекст обработчика события][6]

### Использование анонимных функций

Мы ожидали, что обработчик события выведет сообщение `Меня зовут Виктор`. На самом деле он выведет `Меня зовут undefined`. Чтобы `this.firstName` возвращало `Виктор`, нужно вызвать `user.greeting` в контексте (т.е. это то, что должно быть слева от точки) `user`.

Когда мы передаём функцию `greeting` методу `addEventListener`, мы всего лишь указываем отношение к функции; контекст `user` с ней не передается. По сути, функция вызвана в контексте `element`, это значит что `this` обозначает `element`, а не `user`. Следовательно, `this.firstname` не может быть определено.

Есть два способа предотвратить такой дисбаланс контекста. Во-первых, можно вызвать `user.greeting()` с правильным контекстом в анонимной функции.

    element.addEventListener('click', function() {
      user.greeting();
      // alert => 'Меня зовут Виктор'
    });

[Демо: Анонимные функции][7]

### `Function.prototype.bind`

Последний метод недостаточно хорош, потому что когда мы хотим удалить функцию с помощью `.removeEventListener()` оказывается что у нас нет к ней доступа. Кроме того, такой подход довольно безобразный. Я предпочитаю использовать метод `.bind()` (встроен во все функции ECMAScript 5) для генерации новой функции (привязанной), которая всегда выполняется в заданном контексте. Затем мы передаём эту функцию методу `.addEventListener()` в качестве функции обратного вызова.

    // Перекрытие исходной функции 
    // функцией, привязанной к контексту 'user'
    user.greeting = user.greeting.bind(user);

    // Присоединение user.greeting в качестве функции обратного вызова
    button.addEventListener('click', user.greeting);

У нас также есть возможность сослаться на функцию обратного вызова, которую можно использовать, чтобы отвязать приемник события при необходимости.

    button.removeEventListener('click', user.greeting);

[Демо: Function.prototype.bind][8]

> Если нужно, можете зайти на [страницу с информацией о поддержке][9] `Function.prototype.bind` и страницу с описанием [полифила][10]. 

## Объект `событие`

Объект `событие` создаётся когда соответствующее событие происходит впервые; он сопровождает событие в пути по дереву документа. Объект `событие` передаётся в качестве первого параметра функции, которую мы прописываем для приемника события как функцию обратного вызова. Этот объект можно использовать чтобы получить доступ к огромному количеству информации о случившемся событии:

* `type` (строка). Это имя события.
* `target` (узел). Это узел DOM, который породил событие.
* `currentTarget` (узел). Это узел DOM, для которого на текущий момент работает обработчик события. 
* `bubbles` (булево значение). Показывает является ли событие «всплывающим» (что это значит, я [объясню позже][11]). 
* `preventDefault` (функция). Позволяет предотвратить любое поведение, установленное по умолчанию, со стороны агента пользователя (т.е. браузера) в отношении события (например, предотвращение загрузки новой страницы вследствие события `click` элемента `<a>`).
* `stopPropagation` (функция). Предотвращает запуск следующих обработчиков дальше по цепочке событий, однако не предотвращает запуск дополнительных обработчиков события с тем же именем для текущего узла. (Мы поговорим об этом [позже][12].)
* `stopImmediatePropagation` (функция). Предотвращает запуск обработчиков для любых узлов дальше по цепочке событий, а также дополнительных обработчиков события с тем же именем для текущего узла. 
* `cancelable` (булево значение). Указывает на то, можно ли с помощью метода `event.preventDefault` предотвратить запуск действий по умолчанию в ответ на событие.
* `defaultPrevented` (булево значение). Указывает был ли вызван метод `preventDefault` для объекта `событие`. 
* `isTrusted` (булево значение). Событие называется доверенным если оно исходит от самого устройства, а не синтезируется в JavaScript.
* `eventPhase` (число). Это число указывает фазу, в которой на данный момент находится событие: ни в какой (`0`), перехват (`1`), цель (`2`) или всплытие (`3`). По фазах мы пройдёмся [дальше][13]. 
* `timestamp` (число). Это дата, когда произошло событие.

Объект `событие` может принимать множество других свойств, однако они зависят от конкретного типа события. Например, для событий мыши для объекта `событие` применяются свойства `clientX` и `clientY` чтобы определить размещение указателя в области просмотра. 

Для более подробного ознакомления с объектом `событие` и его свойствами лучше всего использовать отладчик в вашем любимом браузере или `console.log`. 

## Фазы события

Когда в приложении возникает событие DOM, оно не просто срабатывает один раз в месте своего происхождения; оно отправляется в путь, состоящий из трёх фаз. Вкратце, событие движется от корня документа к цели (фаза перехвата), затем срабатывает для цели события (фаза цели) и движется назад к корню документа (фаза всплытия).

![поток события][поток события]

*(Источник изображения: [W3C][14])*

[Демо: Замедленное воспроизведение продвижения события][15]

### Фаза перехвата

Первой фазой является фаза перехвата. Событие начинает своё продвижение в корне документа, проходит каждый слой дерева документа по направлению к цели, срабатывая для каждого узла пока её не достигнет. Задача фазы перехвата - наметить траекторию распространения, по которой событие будет двигаться в обратном направлении в фазе всплытия. 

Как упоминалось ранее, обработчик можно вызвать в фазе перехвата если в качестве третьего параметра для `addEventListener` указать `true`. Я особо не сталкивался с ситуациями когда требовался вызов обработчика в фазе перехвата, однако теоретически с его помощью можно предотвратить срабатывание события щелчка мышью для определённого элемента если событие обрабатывается в фазе перехвата. 

    var form = document.querySelector('form');

    form.addEventListener('click', function(event) {
      event.stopPropagation();
    }, true); // Заметьте: 'true'

Если вы не уверены в позитивном результате, вызывайте обработчик события в фазе всплытия, указав для `useCapture` значение `false` или `undefined`.

### Фаза цели

Момент, когда событие достигает конечного объекта, известен как фаза цели. Событие срабатывает для целевого узла перед тем, как изменить направление своего продвижения и вернуться назад к самому внешнему уровню документа. 

В случае с вложенными элементами, события мыши и указателя мыши всегда нацелены на наиболее глубоко расположенный вложенный элемент. Если обработчик вызывается для события `click` элемента `<div>`, и пользователь кликает по элементу `<p>` внутри `<div>`, этот `<p>` становится целью события. Тот факт, что события «всплывают» значит что можно вызывать обработчик для кликов по `<div>` (или любому другому родительскому элементу) и получать функцию обратного вызова при прохождении события.

### Фаза всплытия

После того, как событие сработало для цели, оно на этом не останавливается. Оно всплывает наверх (или же распространяется) по дереву документа, пока не достигнет его корня. Это значит, что то же самое событие срабатывает для родительского узла целевого элемента, затем для его родительского узла, и это продолжается пока не остаётся родительских элементов, для которых может сработать событие. 

Можно представить дерево документа в виде луковицы, а цель события - в виде её сердцевины. В фазе перехвата событие проникает внутрь луковицы, минуя слой за слоем. Когда событие достигает сердцевины, оно срабатывает (фаза цели) и меняет направление на противоположное, слой за слоем прокладывая себе путь назад (фаза распространения). 

Всплытие события - это очень полезное явление. Оно делает необязательным установку обработчика события для конкретного элемента, для которого происходит событие; вместо этого мы можем установить обработчик для элемента выше по дереву документа и подождать пока событие его достигнет. Если бы события не всплывали, нам, возможно, в некоторых случаях пришлось бы устанавливать обработчики для множества разных элементов для гарантии, что событие не останется незамеченным.

[Демо: Определение фаз события][16]

Большинство (но не все) событий всплывают. Если событие не всплывает, для этого наверняка есть веская причина. В случае сомнения по этому поводу, можете свериться со [спецификацией][16]. 

## Останавливаем распространение

Прервать путь распространения события в любой его момент (т.е. в фазе перехвата или всплытия) можно просто вызвав метод `stopPropagation` объекта `событие`. После этого событие не будет вызывать обработчики для узлов, которые он минует на своём пути к цели события и назад к корню документа.

    child.addEventListener('click', function(event) {
     event.stopPropagation();
    });

    parent.addEventListener('click', function(event) {
     // Если произошёл клик мышью по дочернему элементу,
     // этот обработчик не будет вызван
    });

Если для одного и того же события установлено несколько обработчиков, вызов `event.stopPropagation()` не предотвратит их срабатывания для текущей цели события. Если нужно предотвратить вызов любых дополнительных обработчиков для текущего узла, можно использовать более агрессивный метод `event.stopImmediatePropagation()`.

    child.addEventListener('click', function(event) {
     event.stopImmediatePropagation();
    });

    child.addEventListener('click', function(event) {
     // Если произошёл клик мишью по дочернему элементу,
     // этот обработчик не будет вызван
    });

[Демо: Остановка распространения][17]

## Предотвращение поведения, установленного в браузере по умолчанию

Для некоторых событий, которые происходят в документе, в браузере установлено поведение по умолчанию. Наиболее распространённым является клик по ссылке. Когда для элемента `<a>` происходит событие `click`, оно всплывает до корня документа, браузер расшифровывает атрибут `href` и перегружает окно с новой страницей. 

В веб-приложениях для обработчиков обычно желательно иметь возможность самостоятельно управлять навигацией, без перезагрузок страницы. Чтобы такую возможность получить, нужно предотвратить установленную по умолчанию реакцию браузера на клик, и вместо неё выполнить то, что задумали мы. Для этого мы вызовем `event.preventDefault()`.

    anchor.addEventListener('click', function(event) {
      event.preventDefault();
      // Выполнение нужных нам действий
    });

Также можно предотвратить множество других установленных по умолчанию действий браузера. Например, можно запретить прокручивание страницы в игре на HTML5 при нажатии клавиши пробела, или предотвратить выделение текста при помощи кликов мышью. 

Если вызвать `event.stopPropagation()`, мы всего лишь избежим вызова обработчиков, установленных для элементов дальше по цепочке распространения. Он не помешает браузеру выполнить свою работу.

[Демо: Предотвращение поведения, установленного по умолчанию][18]

## Пользовательские события

Запустить событие DOM может не только браузер. Мы можем создать собственное пользовательское событие и применить его к любому элементу в документе. Событие такого типа ведёт себя точно так же как обычное событие DOM.

    var myEvent = new CustomEvent("myevent", {
      detail: {
        name: "Виктор"
      },
      bubbles: true,
      cancelable: false
    });

    // Вызов обработчика для события 'myevent'
    myElement.addEventListener('myevent', function(event) {
      alert('Привет, ' + event.detail.name);
    });

    // Запуск события 'myevent'
    myElement.dispatchEvent(myEvent);

Также для имитации пользовательского взаимодействия можно синтезировать «не доверенные» события элементов (например, `click`). Они могут пригодиться при тестировании библиотеки для DOM. Если вас это заинтересовало, проект Mozilla Developer Network предлагает [описание работы с такими событиями][19].

Помните следующее:

* `CustomEvent` API не доступен в IE 8 и старше.
* В фреймворке [Flight][20] для Twitter пользовательские события используются для обмена данными между модулями. Такой подход привёл к крайне несвязной модульной архитектуре.

[Демо: Пользовательские события][21]

## Делегированный обработчик события

Применение делегированного обработчика события является удобным и производительным способом обработки события большого количества узлов DOM при наличии одного обработчика. Например, если в списке есть 100 пунктов и они все должны реагировать на событие `click` одинаково, мы можем для каждого из них установить обработчик события. Это даст нам 100 отдельных обработчиков события. При каждом добавлении пункта в список нужно было бы устанавливать для него обработчик события `click`. Это не только дорого, но и неудобно.

Делегированные обработчики событий могут значительно упростить нам жизнь. Вместо того чтобы вешать обработчик события `click` на каждый элемент, мы устанавливаем только один для родительского элемента `<ul>`. После клика по `<li>` событие всплывает до `<ul>` и возбуждает обработчик события. По какому именно элементу `<li>` был произведён клик можно определить проверив `event.target`. Ниже в качестве иллюстрации приведён грубый пример:

    var list = document.querySelector('ul');

    list.addEventListener('click', function(event) {
      var target = event.target;
 
      while (target.tagName !== 'LI') {
        target = target.parentNode;
        if (target === list) return;
      }

      // Выполнение каких-то действий
    });

Такой подход лучше, потому что мы имеем дело лишь с одним обработчиком события и не вынуждены обременять себя установкой новых обработчиков при каждом добавлении новых пунктов списка. Концепция довольно простая, но на удивление полезна.

В приложении такую грубую реализацию я бы не советовал использовать. Лучше обратиться к библиотекам JavaScript для делегирования событий, таким как [ftdomdelegate][22] от команды FT Labs. Если вы используете jQuery, то можете применить её встроенную возможность делегирования событий передавая методу `.on()` селектор в качестве второго параметра. 

    // Без делегирования события
    $('li').on('click', function(){});

    // Использование делегирования события
    $('ul').on('click', 'li', function(){});

[Демо: Делегированный обработчик события][23]

## Полезные события

### `load`

Событие `load` происходит при окончании загрузки любого ресурса (в том числе зависимых ресурсов). Им может быть изображение, таблица стилей, скрипт, видео, аудио файл, документ или окно. 

    image.addEventListener('load', function(event) {
      image.classList.add('has-loaded');
    });

[Демо: Событие `load` объекта `изображение`][24]

### `onbeforeunload`

`window.onbeforeunload` даёт разработчикам возможность запросить у пользователя подтверждение намерения покинуть страницу. Это может пригодиться в приложениях, в которых изменения должны быть сохранены пользователем, иначе будут потеряны при закрытии вкладки браузера. 

    window.onbeforeunload = function() {
      if (textarea.value != textarea.defaultValue) {
        return 'Вы хотите покинуть страницу и отменить изменения?';
      }
    };

Обратите внимание, что установка обработчика `onbeforeunload` не позволяет браузеру [кешировать страницу][25], что приводит к более длительной загрузке страницы при повторном посещении. Кроме того, обработчики `onbeforeunload` должны быть синхронизированы.

[Демо: onbeforeunload][26]

### Избавление от подрагивания окна в Mobile Safari

В приложении Financial Times мы используем простой приём `event.preventDefault` для избежания подрагивания окна при прокрутке в Mobile Safari.

    document.body.addEventListener('touchmove', function(event) {
     event.preventDefault();
    });

Стоит также знать, что это заблокирует любое встроенное прокручивание (такое как `overflow: scroll`). Чтобы разрешить встроенное прокручивание для набора элементов, которые в нём нуждаются, следует установить обработчик для того же события элемента, который должен прокручиваться, и установить флаг для объекта события. В обработчике на уровне документа мы решаем нужно ли предотвратить действие по умолчанию для события касания, исходя из наличия флага `isScrollable`.

    // Ниже по дереву мы устанавливаем флаг
    scrollableElement.addEventListener('touchmove', function(event) {
     event.isScrollable = true;
    });

    // Выше по DOM проверяем наличие этого флага чтобы определить
    // нужно ли разрешить браузеру выполнить прокручивание
    document.addEventListener('touchmove', function(event) {
     if (!event.isScrollable) event.preventDefault();
    });

В IE 8 и старше нельзя управлять объектом события. Чтобы обойти эту проблему можно установить свойства для узла `event.target`.

### `resize`

Возможность установить обработчик для события `resize` объекта `window` ужасно удобна при сложной отзывчивой верстке страницы. Добиться такой верстки только с помощью CSS не всегда возможно. Иногда приходится использовать JavaScript для расчёта и применения размера элементов. Когда изменяется размер окна или же ориентация устройства, нам нужно подстроить эти размеры.

    window.addEventListener('resize', function() {
      // обновление верстки
    });

Я рекомендую использовать обработчик [с ограничением количества вызовов][27] чтобы нормализировать частоту вызова обработчика и избежать слишком частого пересчёта верстки.

[Демо: изменение размеров окна][28]

### `transitionEnd`

Сегодня для реализации большинства переходов и анимации в наших приложениях мы используем CSS. Однако иногда нам все же нужно узнать когда выполнение определённой анимации подошло к концу.

    el.addEventListener('transitionEnd', function() {
     // Выполнение каких-либо действий
    });

Обратите внимание на следующее:

* При создании анимации с применением свойства `@keyframe`, используйте в качестве имени события `animationEnd`, а не `transitionEnd`.
* Как и большинство событий, `transitionEnd` всплывает. Помните что нужно вызвать `event.stopPropagation()` для дочерних событий перехода, либо проверить `event.target` чтобы предотвратить обработку событий, когда она не нужна.
* Для большинства имен событий всё еще широко используются вендорные префиксы (например, `webkitTransitionEnd`, `msTransitionEnd`, и т.д.). Используйте библиотеку вроде [Modernizr][29] для получения имен с правильными вендорными префиксами. 

[Демо: Окончание перехода][30]

### `animationiteration`

Событие `animationiteration` происходит при завершении каждой итерации для текущей анимации элемента. Оно пригодится когда нужно остановить анимацию, но только не в разгаре воспроизведения.

    function start() {
      div.classList.add('spin');
    }

    function stop() {
      div.addEventListener('animationiteration', callback);

      function callback() {
        div.classList.remove('spin');
        div.removeEventListener('animationiteration', callback);
      }
    }

Если вы заинтересовались, я [написал о событии `animationiteration`][31] более подробно в своём блоге. 

[Демо: итерации анимации][32]

### `error`

Если в процессе загрузки ресурса происходит ошибка, нам наверняка нужно на это как-то отреагировать, особенно если у наших пользователей плохое соединение с сетью. В приложении Financial Times используется событие `error` для определения изображений, которые не удалось загрузить в статье и их немедленного скрытия. Так как согласно спецификации «[DOM Уровень 3 События (DOM Level 3 Events)][33]» событие `error` всплывать не должно, обработать его можно одним из двух способов.

    imageNode.addEventListener('error', function(event) {
      image.style.display = 'none';
    });

К сожалению, `addEventListener` подходит не во всех случаях. Мой коллега [Kornel][34] любезно представил мне [пример, который доказывает][35] что, к сожалению, единственный способ гарантировать вызов обработчика события `error` для изображения состоит в использовании строчных обработчиков события (хоть они и не приветствуются).

    <img src="http://example.com/image.jpg" onerror="this.style.display='none';" />

Причиной этому является то, что нельзя быть уверенным что код, с помощью которого установлен обработчик события `error`, будет выполнен до того, как собственно произойдёт событие `error`. Использование строчного обработчика значит что наш обработчик события `error` будет установлен при обработке разметки и запросе изображения.

[Демо: Ошибка загрузки изображения][36]

## Выводы

Из успешности концепции событий DOM можно сделать определённые выводы. Мы можем использовать похожие концепции в наших собственных проектах. Модули в приложении могут быть настолько сложными, насколько это нужно, пока это не сказывается на простоте интерфейса. Большое количество фрронтенд фреймворков (таких как Backbone.js) в большей мере основаны на событиях. 

**Архитектура, построенная на событиях великолепна**. Она даёт нам простой и понятный интерфейс, в котором можно писать приложения, отзывчивые к физическим взаимодействиям на тысячах устройств! Посредством событий устройства с точностью говорят нам что произошло и когда, давая возможность отреагировать так, как мы пожелаем. То, что происходит «под капотом», нас не волнует; мы получаем тот уровень абстрагирования, который даёт нам полную свободу для создания великолепного приложения.

### Материалы для дальнейшего чтения

* “[Спецификация: Объектная модель документа уровень 3, События][37],” W3C
* “[Графическое представление распространения события по дереву документа с использованием потока события DOM][38]” (изображение) W3C
* [“Событие][39],” Mozilla Developer Network
* “[Хитрости в структуре DOM II][40],” Дж. Дэвид Айзенберг (J. David Eisenberg), A List Apart
* “[Таблицы совместимости событий][41],” Quirksmode

*Особую благодарность выражаю [Kornel][42] за отличный технический анализ*.

[1]: http://jquery.com/
[2]: https://developer.mozilla.org/en-US/docs/Web/API/EventTarget.removeEventListener#Polyfill_to_support_older_browsers
[3]: http://jquery.com/
[4]: http://jsbin.com/ayatif/2/edit
[5]: http://jsbin.com/ayamur/1/edit
[6]: http://jsbin.com/atoluy/1/edit
[7]: http://jsbin.com/onomud/1/edit
[8]: http://jsbin.com/ozolec/1/edit
[9]: http://kangax.github.io/es5-compat-table/#Function.prototype.bind
[10]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function/bind#Compatibility
[11]: http://coding.smashingmagazine.com/2013/11/12/an-introduction-to-dom-events/#bubble-phase
[12]: http://coding.smashingmagazine.com/2013/11/12/an-introduction-to-dom-events/#stopping-propagation
[13]: http://coding.smashingmagazine.com/2013/11/12/an-introduction-to-dom-events/#event-phases
[14]: http://www.w3.org/TR/DOM-Level-3-Events/#event-flow
[15]: http://jsbin.com/exezex/4/edit
[16]: http://www.w3.org/TR/DOM-Level-3-Events/#event-types
[17]: http://jsbin.com/aparot/3/edit
[18]: http://jsbin.com/ibotap/1/edit
[19]: https://developer.mozilla.org/en-US/docs/Web/Guide/API/DOM/Events/Creating_and_triggering_events?redirectlocale=en-US&redirectslug=Web%2FGuide%2FDOM%2FEvents%2FCreating_and_triggering_events#Triggering_built-in_events
[20]: http://flightjs.github.io/
[21]: http://jsbin.com/emuhef/1/edit
[22]: https://github.com/ftlabs/ftdomdelegate
[23]: http://jsbin.com/isojul/1/edit
[24]: http://jsbin.com/uhimir/1/edit
[25]: https://developer.mozilla.org/en-US/docs/Using_Firefox_1.5_caching
[26]: http://jsbin.com/inelaj/2/edit
[27]: http://davidwalsh.name/function-debounce
[28]: http://jsbin.com/usevow/1/edit
[29]: http://modernizr.com/
[30]: http://jsbin.com/ijogok/1/edit
[31]: http://wilsonpage.co.uk/animation-iteration-event/
[32]: http://jsbin.com/AwoYuxE/2
[33]: http://www.w3.org/TR/DOM-Level-3-Events/#event-type-error
[34]: https://twitter.com/pornelski
[35]: http://jsbin.com/esimAWA/2/quiet
[36]: http://jsbin.com/ekulop/2/edit
[37]: http://www.w3.org/TR/DOM-Level-3-Events/
[38]: http://www.w3.org/TR/DOM-Level-3-Events/#dom-event-architecture
[39]: https://developer.mozilla.org/en/docs/Web/API/Event
[40]: http://alistapart.com/article/domtricks2
[41]: http://www.quirksmode.org/dom/events/
[42]: https://twitter.com/pornelski

[поток события]: img/eventflow-ru.png