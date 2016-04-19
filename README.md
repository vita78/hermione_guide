# Руководство по разработке hermione-тестов

Документ содержит описание общих сведений о тестировании, основных действий для разрабоки hermione-тестов и вариантов их запуска, в зависимости от сервиса (СЕРП, Мультимедиа).

- Общие сведения
- Основные понятия
- [Конфигурация среды тестирования](#setup-env)
- [Расположение тестов в файловой системе проекта](#fs)  
- [Запуск тестов](#run-tests)
  - [в удаленном гриде](#run-loc)
  - [локальном гриде](#run-remout)
  - [в TeamCity](#run-tc)
- [Отчеты](#reports)
- [Этапы разработки тестов](#steps)
- [Миграция с тестов java на javascript](#migration)  
- [Cоздание тестов](#create-test)  
- [Генерация тестовых данных](#get-test-data)
- FAQ
- [Практики разработки](https://github.yandex-team.ru/mm-interfaces/fiji/blob/beste2epractices/e2e/readme.md)

## Общие сведения и основные понятия

`hermione-тесты` — это [интеграционные тесты](http://www.intuit.ru/studies/courses/1040/209/lecture/5412), которые разрабатываются и запускаются с помощью инструмента для автоматизации тестирования [Hermione](https://github.com/gemini-testing/hermione). В его основе лежат [WebdriverIO](http://webdriver.io/) и [Mocha](https://mochajs.org) с рядом дополнительных возможностей, упрощающих тестирование: параллельный запуск тестов, повторный запуск непройденных тестов, подключение плагинов и многое другое.

Тестирование функциональности сервиса заключается в проверке работы его бизнес-логики и взаимодействия компонентов (наборов блоков) в соответствии с ожидаемым результатом на основе тестовых данных. Выполняется путем эмуляции пользовательского ввода (сценария) в браузере и получения предпологаемого результата.

Довольно часто такие тесты могут состоят из переходов по нескольким страницам, ввода данных в различные формы и поля, операции подтверждения и отмены действий и проверки результатов. Обычно такими тестами выполняется проверка процессов входа в систему, изменения настроек, сложных операций по получению данных, и тому подобного.  

Для процесса тестирования основными элементами являются:

- **Тестовые сценарии** (test case) — это описание начальных условий, входных данных, последовательность
и действий пользователя и ожидаемого выходного результата. Необходимы для проверки соответствия тестируемой функция установленным требованиям.

- **Наборы тестовых сценариев** (test suite) — тестовые сценарии, объединенные в группу по некоторому признаку, например, тестируемой функциональности. Они могут быть как зависящими от последовательности выполнения (данные, полученные в ходе  выполнения предыдущего теста являются предварительным условием для следующего), так и независимыми.

- **Тестовые данные** (test data) — это фиксированный набор значений, который используется для выполнения тестов на изменяющихся входных данных с заранее известным результатом. Они не зависят от внешних источников и могут повторно использоваться для оценики правильности выполнения тестов (то есть служат эталоном с которым сравниваются результаты тестов).


<a name="setup-env"></a>
## Конфигурация среды тестирования

Для разработки и выполнения тестов, помимо утилиты `Hermione`, могут понадобиться следующие инструменты:

* [Selenium Server](https://github.com/vvo/selenium-standalone/blob/master/README.md) —  сервер, для запуска тестов только в локальном [Selenium Grid](https://wiki.yandex-team.ru/selenium/).

* Кэширующий сервер —  NodeJS-cервер, для локальной разработки и тестирования проекта в зависимости от сервиса :
  * [node-report](https://github.yandex-team.ru/mm-interfaces/fiji/blob/dev/README.md#node-report) (Мультимедиа).
  * [templar](https://github.yandex-team.ru/nodereport/templar) (СЕРП).
  > **NB** Подключен в виде [плагина](https://github.com/gemini-testing/hermione#plugins) [hermione-templar](https://github.yandex-team.ru/nodereport/hermione-templar), который создает SSH-туннель и запускает `templar` автоматически. Подключается в конфиге `Hermione` ([Пример](https://github.yandex-team.ru/serp/web4/blob/dev/.hermione.conf.js#L28) подключения в `web4`).
* [GitLFS](https://wiki.yandex-team.ru/search-interfaces/testing/layout/git-lfs/#ustanovka) — клиент Git, который организует хранилище больших файлов и позволяет хранить в репозитории не сами файлы, а ссылки на них. Используется для хранения тестовых данных (используется только в `web4`).

* [Hermione](https://github.com/gemini-testing/hermione) — утилита, для разработки интеграционных тестов.  
Рекомендуется локальная установка из корня проекта: `npm install hermione`. ???
Конфигурируется в файле `.hermione.conf.js`, расположенном в корне проекта. Он должен содержать, как минимум, две обязательные секции:
  * [specs](https://github.com/gemini-testing/hermione#specs) — список путей к папкам с  hermione-тестами.
  * [browsers](https://github.com/gemini-testing/hermione#browsers) — список браузеров для запуска тестов.

Подробнее читайте в разделе документации [Quick Start](https://github.com/gemini-testing/hermione#quick-start).
>**СЕРП**  

Обратите внимание, что в `package.json` проекта `web4` нет зависимостей от `templar` или `hermione`. Необходимые пакеты нужно установить самостоятельно:

```
 web4$ cd hermione && npm i --production && cd ..
```
**Важно!**  Если нужно добавить пакеты для hermione-тестов, добавьте их в `web4/hermione/package.json` и НЕ добавляйте в `web4/package.json`.

<a name="fs"></a>
### Расположение тестов и тестовых данных в cтруктуре проекта

(Вариант подзаголовка: Структура тестов и размещение в проекте)

Тестовые сценарии хранятся в файлах с расширением `*.js` в  папках, указаных в секции `specs` конфигурационного файла `.hermione.conf.js`.

Наборы тестовых данных, на основе которых выполняются тестовые сценарии, как правило, хранятся отдельно от тестов во внешних файлах или загружаются из базы данных. Способ их хранения и расположение зависит от сервиса и настроек его кэширующего сервера.
Тестовые данны могут повторно использоваться при проверке выполнения тестов как при локальной разработке тестов, так и при удаленном запуске.

Подробнее про подготовку тестовых данных читайте в разделе [Генерация тестовых данных](#get-test-data).

> **СЕРП**

В проекте [web4](https://github.yandex-team.ru/serp/web4) тесты и их данные хранятся в отдельной папке `hermione`:

* тесты расположены в папке `hermione/test-suites/<уровень_ переопределения>`;
* данные расположены в папке `hermione/test-data/<id-теста_из_ TestPalm>/<уровень_переопределения>`, хранятся с помощью [GitLFS](https://wiki.yandex-team.ru/search-interfaces/testing/layout/git-lfs/#ustanovka).

> **Мультимедиа**

В проекте [fiji](https://github.yandex-team.ru/mm-interfaces/fiji) :
* тесты хранятся в папке вида `e2e/<название_сервиса>/<уровень_переопределения>`,например, `e2e/images/desktop/search.js`.
* данные хранятся в специальном репозитории [report-cache]( https://github.yandex-team.ru/mm-interfaces/report-cache).

<a name="run-tests"></a>
## Запуск тестов

Способы запуска тестов:

* [Локальный запуск в удаленном Selenium Grid](#run-remout);
* [Локальный запуск в локальном Selenium Grid](#run-loc);
* В ТeamСity в удаленном Selenium Grid при сборке PR.

Быстрая справка c кратким описанием опций:

```
hermione/node_modules/.bin/hermione --help
```

<a name="run-remout"></a>
### Запуск в удаленном гриде

```bash
# Запуск тестов во всех браузерах
hermione/node_modules/.bin/hermione --play

# Запуск тестов в одном браузере
hermione/node_modules/.bin/hermione --play -b firefox

# Запуск одного тестового файла

hermione/node_modules/.bin/hermione  <путь_к_файлу_теста>/file.js
```

Чтобы запустить отдельный набор тестов `describe`, ему в коде теста нужно добавить ключ `only`:

```bash
# Запуск определенного набора тестов в пределе одного файла
describe.only('some tests, function() {
    it('some test');
    it('some test2');
});
```

```bash
# Запуск определенного теста в пределе одного набора тестов `describe`
describe('some tests, function() {
    it.only('some test');
    it('some test2');
});
```
 **NB** Список браузеров для запуска тестов определяетcя в секции `browsers` файла `.hermione.conf.js`.

Для запуска hermione-тестов в `fiji` используетя обертка над командой `hermione` — команда `e2e` .

> **CЕРП**

```bash
# Запустить только десктопные тесты
PLATFORM=desktop hermione/node_modules/.bin/hermione --play

# Запустить только десктопные тесты, только в одном браузере
PLATFORM=desktop hermione/node_modules/.bin/hermione --play -b firefox
```

**Внимание** Необходимо обязательно указывать платформу для запуска тестов в переменной `PLATFORM`. Иначе, тесты будут выполняться во всех доступных браузерах грида (мобильных, десктопных). Это приведет к падению тестов, предназначенных только для определенной платформы.

Допустимые значения переменной `PLATFORM`: `desktop`, `touch-pad`, `touch-phone` (указывать можно только одно из значений).

>**Мультимедиа**

```bash
# Запуск тестов в удаленном Selenium Grid, выполняется с использованием опции `--remote-grid`
fiji e2e e2e/mocha/images/desktop  --browser desktop-chrome --remote-grid --url yandex.ru
```

<a name="run-loc"></a>
### Запуск в локальном гриде

Если для разработки и отладки тестов нет возможности запуска их на [внутреннем Selenium Grid](https://selenium.yandex-team.ru/), используйте локальный Selenium Grid.

Перед первым запуском тестов нужно:

* Установить [selenium-standalone](https://www.npmjs.com/package/selenium-standalone selenium-standalone):

```
npm i -g selenium-standalone
selenium-standalone install
```

Если произошла ошибка, установить Java, которая требуется для работы `selenium-standalone`. Cкачайте со [страницы](http://www.oracle.com/technetwork/java/javase/downloads/jre7-downloads-1880261.html) файл `jre-7u79-macosx-x64.dmg` и установите его. Затем добавьте в `PATH`, например так: `export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home`.

Перед каждым запуском тестов следует:

* Запустить `selenium-standalone` в **отдельной** вкладке c помощью команды:

```
selenium-standalone start
```

> **СЕРП**

В команде запуска тестов указать адрес локального грида с помощью параметра `--grid <url хаба>`:

```
hermione/node_modules/.bin/hermione --play --grid http://127.0.0.1:4444/wd/hub`
```

URL хаба можно найти в логе запущенного `selenium-standalone` в строчке вида: `15:18:16.753 INFO - RemoteWebDriver instances should connect to: http://127.0.0.1:4444/wd/hub`.

** NB** Для локального запуска нужно отключить SSH-туннель. Для этого установите полю `tunnel.enabled` значение `false` в секции 'hermione-tunnel' файла `.hermione.conf.js`.

> **Мультимедиа**

По умолчанию тесты запускаются в локальном Selenium Gride через SSH-тунель, который  до прокси создается автоматически и тесты выполняются в нужных браузерах, в зависимости от тестируемой платформы.

```bash
# Запуск всех тестов (по умолчанию локальный)
fiji e2e [options] [test]  # `tests` — путь к тесту;  `options` — параметры запуска

# Доступные параметры:
fiji e2e --help

```

```bash
# Запуск тестов в определенном браузере:
fiji e2e <путь к тестам> --browser <браузер> --url <бета, которую тестируем>
Например:
fiji e2e --browser=desktop-chrome --url=dev.local.yandex.ru

# Запуск одного или нескольких тестов, выполняется указанием пути к файлу или директории с тестами:
fiji e2e e2e/mocha/images/desktop/misspell.js --browser=desktop-firefox
```

<a name="report"></a>
## Отчеты

Результаты выполнения тестов можно просмотреть несколькими способами:

* Локально в консоли.
* Удаленно в конфигурации TeamCity (найти можно по номеру PR) с помощью:
   * `teamcity reporter`, отчет доступен на вкладке `Tests`.

![Картинка](file:hermione-report.jpg)

   * `allure reporter`, oтчет доступен на вкладке `Hermione Allure Report`.

![Изображение](https://jing.yandex-team.ru/files/gavryushin/SI%20%3A%3A%20SERP%20%3A%3A%20web4%20%3A%3A%20Pull%20requests%20%3A%3A%202.%20Hermione%20%3A%3A%20desktop%20>%20%234560%20%2803%20Mar%2016%2017%3A10%29%20>%20Overview%20—%20TeamCity%202016-03-03%2017-15-08.png)

<a name="steps"></a>
## Основные этапы разработки тестов

 - Разработчик, тестировщик и менеджер совместно продумывают и состовляют словесное пошаговое описание действий пользователя для тестируемой функциональности (для каждой платформы). Затем, сохраняют его в текстовом виде способом, принятым на сервисе. Например, в файле репозитория, рядом с кодом или в системе хранения тестовых сценариев.
В СЕРПе сценарии хранятся в системе [TestPalm](https://testpalm.yandex-team.ru/serp) в секции «Steps» (см.[пример](https://testpalm.yandex-team.ru/testcase/serp-2458)).
- Менеджер и тестировщик описывает сценарии по принципу «Что дано/Что сделали/Что Получили».
[Пример сценария](https://st.yandex-team.ru/SERP-39936#1455118360000)
- Разработчик кодирует и реализует тестовые сценарии.
- Тестировщик проверяет код тестов.

<a name="migration"></a>
### Миграция тестов с Java на JavaScript

- Разработчик создает задачу в ST для переработки автотеста на Java и переносит в нее  `id` теста  и описание сценария (секции «Steps») из [TestPalm](https://testpalm.yandex-team.ru/serp).
- Определяет какие действия в тесте можно проверить с помощью `hermione`-тестов, а какие с помощью `gemini`-тестов (визуальная составляющая).
- В комментариях задачи переписывает часть описания теста, которая будет тестироваться с промощью `hermione`, в терминах приближенных к тестовым сценариям в виде цепочки шагов «действие -> ожидаемый результат» [см. пример](https://st.yandex-team.ru/SERP-39936#1455118360000)
- Кодирует описанные действия c использованием `Mocha` и `Webdriver.io+Chai`.
- Тестировщик проверяет код созданных `hermione-` и `gemini-`тестов на полноту и правильность выполнения.

<a name="create-test"></a>
### Cоздание тестового сецнария

Для разработка и запуска тестов используются следующие библиотеки:

- [Mocha](https://mochajs.org) — тестовый фреймворк.
- [WebdriverIO](http://webdriver.io/) — утилита, обеспечивающая взаимодействие с браузером.
- [Chai](http://chaijs.com/) — библиотека ассертов (в конфиге `hermione.conf.js` можно подключить любую библиотеку ассертов).

Рассмотрим поэтапную разработку тестов на простом примере появления колдунщика погоды:  

1. Описываем текстом последовательность действий пользователя для появления ожидаемого результата:
```
Пользователь в инпут ввел текст: погода в симферополе
Нажал кнопку «Найти»
Должен появиться колдунщик погоды
```

1. Создаем тестовый сценарий `test.js` c использованием `WebdriverIO` и `Chai` (в папке, указанной в секции `specs` конфига `hermione.conf.js`):

   **NB** Элементы интерфейса определяются по селектору.

```js
describe('weather', function() {
    it('должен появиться колдунщик погоды', function() {
        return this.browser
            .setValue('.search2__input', 'погода в симферополе')
            .click('.search2__button')
            .waitForVisible('.z-weather', 2000);
    });
});
```

Если при запуске теста (`hermione/node_modules/.bin/hermione test.js`) никаких ошибок не появилось, то тест пройден успешно.

Чтобы упростить поддержку написанных тестов и cократить количество дублируемого кода, можно использовать известный в автоматизированном тестировании шаблон проектирования [Page Object](http://selenium2.ru/docs/test-design-considerations.html#page-object).

 <a name="get-test-data"></a>
## Генерация тестовых данных

В ходе тестирования функциональности тестовые сценарии циклично используют значения из набора тестовых данных — закешированные данные (дамп), хранящиеся в файловой системе. Они были полученны при выполненни тестов на известных входных данных с ожидаемым результатом в в ходе запроса к удаленным источника сервиса, на этапе разработки теста.

Кэш данных применяется для формирования тестовой страницы, на которой выполняются тесты.

 Чтобы получить дамп тестовых данных, нужно на этапе разработки тестов локально, выполнить следующие действия:

> Сервисы используют разные кэширующию сервера (репорт):
* [templar](https://github.yandex-team.ru/nodereport/templar) (СЕРП).
* [node-report](https://github.yandex-team.ru/mm-interfaces/fiji/blob/dev/README.md#node-report) (Мультимедиа).

> СЕРП

- Запустить все текущие тесты проекта `web4` с опцией  репорта `--play` (на основе уже сохраненных ответов репорта).  Убедиться, что все тесты завершились успешно. Если есть упавшие тесты, необходимо устранить ошибки.

```
hermione/node_modules/.bin/hermione --play
```
- Написать код теста и убедиться, что он выполняется правильно, запустив его без опций (по живым данным репорта)

```
hermione/node_modules/.bin/hermione hermione/test-suites/desktop/mytest.js
```

- Сохранить дампы данных своих тестов в файл посредством опции `--save`. Изменения в дампах чужих тестов откатить c помощью опции `--grep` (фильтрует тесты, по строкам или регулярным выражениям).

Чтобы сохранит данные только для тестов отмеченных `only` ( `decribe.only` или `it.only`) нужно запустить тесты с
c опцией `--save` и сохранить данные только для этих тестов.

- Запустить повторно все тесты с опцией `--play`. Убедиться, что ваши тесты  выполняются правильно поверх сохраненных данных (в консоли не должно быть ошибок). Желательно несколько раз, особенно, если в тесте есть AJAX-запросы.   

- Если все тесты прошли успешно, нужно закоммитить `test-suite` и `test-data` своих тестов в репозиторий проекта. Учитывайте, что `test-data` коммитятся в [GitLFS]().

- В отчетах для вашего PR на GitHub-е проверьте, что ваши тесты в TeamCity прошли успешно.

> **Мультимедия**

- Запустить [node-report] в нужном режиме:
 * `read` — чтение значений из файловый системы (по умолчанию)
 * `create` — создание новых ключей. Используется для создания новых ключей и данных для них. Если ключ уже существует, то данные не будут обновлены.
 *  `update` 
