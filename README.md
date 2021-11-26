# QuikQtBridge
Сервер для использования низкоуровневого lua api QUIKа удалённо + новый высокоуровневый апи

Сейчас позволяет маппить весь луа-интерфейс квика в удалённое приложение через низкоуровневый апи посредством
нескольких запросов.

Вообще протокол разбит на два уровня - сеансовый и прикладной. Весь обмен ведётся при помощи Json объектов.

На сеансовом уровне есть всего 4 вида сообщений, задаваемые ключом type:

- req - request, запрос
- ans - answer, ответ
- ver - version, версия
- end - окончание сеанса связи

На сеансовом уровне у каждого сообщения есть id - обычно задаётся запросам на прикладном уровне и обычно ответ имеет тот же id, что и запрос. Версия нужна для прикладного уровня чтобы разруливать несовместимости. Введена на будущее, пока она всегда
будет 1-й.

Последнее поле сеансового уровня data - тут передаются сообщения прикладного уровня.
Ниже приведён пример запроса:

`{"id":2,"type":"req","data":{"method":"invoke","function":"getClassesList","arguments":[]}}`

... и ответа:

`{"id":2,"type":"ans","data":{'method': 'return', 'result': ['PSAU,SMAL,INDX,TQBR,TQOB,TQIF,TQTF,TQOD,CETS,CROSSRATE,SPBFUT,SPBOPT,USDRUB,RTSIDX,REPORT,REPORTFORTS,TQTD,SPBXM,EQRP_INFO,TQTE,TQIE,TQPI,FQBR,FQDE,QT_EQ,QT_BN,EES_CETS,SPBDE,TQFD,TQFE,TQCB,TQOE,TQRD,TQUD,TQED,TQIR,TQIU,']}}`

На прикладном уровне обычно (но не обязательно) присутствует поле method. Для низкоуровневого апи (прямого маппинга а апи луа) там есть следующие методы:

**register** - регистрация колбека

**invoke** - вызов метода квика. Если при этом ещё определён параметр object, то это вызов метода объекта, как в случае с источниками данных, создаваемыми запросом CreateDataSource 

**delete** - удаление ссылки на квиковый объект. Удаляется именно ссылка, сам объект должен удаляться вызовом соответствующего метода объекта (например Close для DataSource)

Вот несколько примеров запросов и ответов:

`{"id":3,"type":"req","data":{"method":"invoke","function":"CreateDataSource","arguments":["TQBR","SBER",5]}}`

`{"id":3,"type":"ans","data":{'method': 'return', 'result': [4]}}`

`{"id":4,"type":"req","data":{"method":"invoke","object":4,"function":"SetUpdateCallback","arguments":[{"type":"callable","function":"sberUpdated"}]}}`

Последний пример это запрос на установку колбека на обновление источника данных. Для этого используется специальный парамер:

`{"type":"callable","function":"sberUpdated"}`

В результате клиент будет получать запросы типа:

`{"id":10,"type":"req","data":{"method":"invoke","function":"sberUpdated","arguments":[15925]}}`

На это клиент обязан обязательно отправить ответ, иначе вызывающий поток квика (тот, что вызвал update callback) будет заморожен (главный поток сервера при этом продолжает работать)

Собственно на этом низкоуровневые методы и заканчиваются - это позволяет делать всё, что можно делать на луа удалённо, из внешней программы.

Высокоуровневых запросов сейчас 7:

**loadAccounts**

`{"id":3,"type":"req","data":{"method": "loadAccounts", "filters": [{"key": "class_codes", "regexp": "SPB"}]}}`

Запрос информации о клиентских счетах. В поле `filters` передаётся список фильтров, которые определяют поле таблицы торговых счетов и регулярное выражение. Все регулярки должны найти совпадение в строке, чтобы она попала в результат. В примере выше запрашиваются все счета, которые имеют доступ к SPB (то есть срочный рынок по сути)

**loadClasses**

`{"id":3,"type":"req","data":{"method": "loadClasses"}}`

Вернёт список классов.

**loadClasSecurities**

`{"id":3,"type":"req","data":{"method": "loadClasSecurities", "class": "TQBR", "filters": [{"key": "lot_size", "regexp": "10"}]}}`

Данный запрос вернёт все бумаги класса TQBR с размером лота = 10 штук. Список полей смотрите в документации quik "4.21 Инструменты"

**subscribeParamChanges и unsubscribeParamChanges**

`{"id":3,"type":"req","data":{"method": "subscribeParamChanges", "class": "TQBR", "security": "SBER", "param": "VOLATILITY"}}`

unsubscribe... выглядит так же, меняется только поле method. Список параметров тот же что при вызове функции lua getParamEx (а где его искать в документации квика я не знаю, но где-то он точно есть :) )

**subscribeQoutes и unsubscribeQuotes**

`{"id":3,"type":"req","data":{"method": "subscribeParamChanges", "class": "TQBR", "security": "SBER"}}`

Это подписка на стаканы, которые будут обновляться автоматом по событию OnQuote и забираться вызовом getQuoteLevel2. Тут есть одна неочевидная приятность - если вы несколько соединений установите и в каждом подпишетесь на один и тот же стакан, то получать оба подписчика будут стаканы по одному событию, дёргая квик один раз.

## Бинарник

Я там добавил каталог bin - там лежит готовая, собраная без зависимостей dll - просто берёте её и кидаете в каталог квика или куда угодно, откуда её сможет загрузить инициализирующий скрипт.

## Инициализирующий скрипт

У меня он выглядит так:

`package.cpath = package.cpath..[[;c:\Work\QuikQtBridge\build-QuikQtBridge-qt_5_15_7_vs2019_static_win64-Debug\?.dll]]`

`require "QuikQtBridge"`

первая строка просто добавляет путь где лежит dll во внутрениий path lua. Вторая строка просто загружает её как библиотеку lua

## Конфигурационный файл

Конфигурационный файл (QuikQtBridgeStart.json) должен лежать в том же каталоге, где и инициализирующий скрипт. Вот так выглядит конфигурационный файл у меня:

`
{
	"host": "anyIPv4",
	"port": 57777,
	"exchangeLogPrefix": "exch",
   	"allowedIPs":[
		"127\\.0\\.0\\.1",
    		"192\\.168\\.0\\.*",
    		"192\\.168\\.2\\.*"
   	]
}
`

host может быть либо конкретным IP (смотрите какие интерфейсы у вас доступны), либо специальными адресами: "local", он же "localhost" - локальный лупбэк, "any" - любой доступный, как IP4 так и IP6, "anyIPv4" и "anyIPv6" - тоже любой, но ограниченный версией IP

port - понятно, но учтите, что номера портов ниже какой-то границы могут быть связаны с какой-то службой и закрыты брандмауэром. Вообще проверить брандмауэр и вирусоловов не помешает - они могут мешать.

exchangeLogPrefix - если этот параметр задан, то к этому префиксу добавится IP и расширение log и в этот файл будет скидываться весь обмен. То есть всё, что принято и отправлено, ничего больше. Если после отладки больше не хотите писать логи, просто удалите этот параметр из конфига.

allowedIPs - список регулярок для проверки IP. Это именно регулярные выражения, поэтому там пишется два слеша - один по синтаксису json, экранирует второй, второй - тот, что будет экранировать в регулярке точку, впрочем вы сами всё знаете :)

