Redis
=====

.. seealso::
  * https://ru.wikipedia.org/wiki/Redis
  * http://eax.me/redis/
  * http://redis.io/

Redis (REmote DIctionary Server) - это нереляционная высокопроизводительная СУБД.

Redis работает на большинстве POSIX систем. Официальной поддержки для сборок Windows нет.

Особенности
-----------

Redis умеет сохранять данные на диск. Можно настроить Redis так, чтобы данные вообще не сохранялись, сохранялись периодически по принципу copy-on-write, или сохранялись периодически и писались в журнал (binlog). Таким образом, всегда можно добиться требуемого баланса между производительностью и надежностью.

В основе своей использует записи вида **ключ-значение**. Ключи бинарнобезопасны. Это значит, что в качесте ключа может быть использована любая бинарная последовательность, полученная хоть из сроки, хоть из JPG-картинки. Максимальный размер ключа - 512 MB.

В качестве значений поддерживаются следующие структуры данных:

* Строки (strings). Это те же самые ключи, но сохранённые как значения.
* Списки (lists). Классические списки строк, упорядоченные в порядке вставки, которая возможна как со стороны головы, так и со стороны хвоста списка.
* Множества (sets). Множества строк в математическом понимании: не упорядочены, поддерживают операции вставки, проверки вхождения элемента, пересечения и разницы множеств.
* Упорядоченные множества (sorted sets). Упорядоченное множество отличается от обычного тем, что его элементы упорядочены по особому параметру "score". Позволяет выбирать группы значений из множества. Например, 10 сверху. Присутствует возможность лексических упорядоченных множеств строк, у которых поле score одинаковое.
* Хеш-таблицы (hashes). Классические хеш-таблицы или ассоциативные массивы.
* Битовые массивы (bitmaps).
* HyperLogLog. Структура данных для реализации алгоритма случайного вероятностного подсчета количества уникальных значений.

Для всех этих типов поддерживаются атомарные операции (например вставка в список или пересечение множеств).

Позволяет хранить не только строки, но и массивы (которые могут использоваться в качестве очередей или стеков), словари, множества без повторов, , а также множества, отсортированные по некой величине. Разумеется, можно работать с отдельными элементами списков, словарей и множеств. Присутствует возможность указать время жизни данных (двумя способами - "удалить тогда-то" и "удалить через ...").

Интересная особенность Redis заключается в том, что это - однопоточный сервер. Такое решение сильно упрощает поддержку кода, обеспечивает атомарность операций и позволяет запустить по одному процессу Redis на каждое ядро процессора. Разумеется, каждый процесс будет прослушивать свой порт.

В Redis есть репликация. Репликация с несколькими главными серверами не поддерживается. Каждый подчиненный сервер может выступать в роли главного для других. Репликация в Redis не приводит к блокировкам ни на главном сервере, ни на подчиненных. На репликах разрешена операция записи. Когда главный и подчиненный сервер восстанавливают соединение после разрыва, происходит полная синхронизация (resync).

Также Redis поддерживает транзакции (будут последовательно выполнены либо все операции, либо ни одной) и пакетную обработку команд (выполняем пачку команд, затем получаем пачку результатов). Притом ничто не мешает использовать их совместно.

С версии 2.6.0 добавлена поддержка Lua, позволяющего выполнять запросы на сервере. Lua позволяет атомарно совершить произвольную обработку данных на сервере и предназначена для использования в случае, когда нельзя достичь того же результата с использованием стандартных команд.

Еще одна особенность Redis - поддержка механизма publish/subscribe. С его помощью приложения могут создавать каналы, подписываться на них и помещать в каналы сообщения, которые будут получены всеми подписчиками. Что-то вроде IRC-чата.

С версии 3.0 появилась встроенная поддержка кластеризации.

Заключение
----------

Redis благодаря его архитектуре очень часто используется как хранилище сессий пользователей, построения очередей, а также для кеша.

Практика
--------

В качестве практики рассмотрим примеры работы с разными структурами данных в Redis.

Настройка окружения
^^^^^^^^^^^^^^^^^^^

Установим необходимые пакеты:

.. code-block:: shell

    wget http://download.redis.io/redis-stable.tar.gz
    tar xzf redis-stable.tar.gz
    cd redis-stable
    make

Запустим сервер:

.. code-block:: shell

    src/redis-server  --loglevel=DEBUG

Подключимся с помощью клиента к запущенному серверу:

.. code-block:: shell

    src/redis-cli

Строки
^^^^^^

Установка / получение значения:

.. code-block:: shell

    > set mykey somevalue
    OK
    > get mykey
    "somevalue"

Через команду ``SET`` можно выполнять условную установку значения для ключа (например, присвоить значение только в том случае, если ключ не / существует), а также задать время "жизни" ключа.

Для целочисленных значений присутствует возможность их де / инкремента через встроенные команды:

.. code-block:: shell

    > set counter 100
    OK
    > incr counter
    (integer) 101
    > incrby counter 50
    (integer) 151
    > decr counter
    (integer) 150
    > decrby counter 50
    (integer) 100

Операции де / инкремента атомарны. Т.е. при чтении двумя клиентов некоторого ключа со значением 10 и последующего его инкремента, финальное значение этого ключа будет 12.

Для присвоения / получения нескольких значений сразу служат команды ``MSET`` и ``MGET`` соответственно:

.. code-block:: shell

    > mset a 10 b 20 c 30
    OK
    > mget a b c
    1) "10"
    2) "20"
    3) "30"

Списки
^^^^^^

Для построения списка необходимо добавить в него элементы. Добавить можно как с головы списка (слева), так и с конца (справа):

.. code-block:: shell

    > lpush mylist A first
    (integer) 2
    > rpush mylist B
    (integer) 3
    > lrange mylist 0 -1
    1) "first"
    2) "A"
    3) "B"

Для получения крайних элементов слева / справа служат соответствующие команды:

.. code-block:: shell

    > lpop mylist
    "first"
    > rpop mylist
    "B"

Существуют блокирующие реализации этих команды ``BLPOP`` и ``BRPOP``. Они вернут результат выполнения только после того как будет добавлен элемент или по истечении времени ожидания.

Хеш-таблицы
^^^^^^^^^^^

Могут служить для хранения каких-либо объектов:

.. code-block:: shell

    > hmset user:1000 username antirez birthyear 1977 verified 1
    OK
    > hget user:1000 username
    "antirez"
    > hget user:1000 birthyear
    "1977"
    > hgetall user:1000
    1) "username"
    2) "antirez"
    3) "birthyear"
    4) "1977"
    5) "verified"
    6) "1"

Аналогично команде HMSET, которая позволяет записать сразу несколько значений в хеш-таблицу, существует команда HMGET, позволяющая получить массив значений из желаемых полей:

.. code-block:: shell

    > hmget user:1000 username birthyear no-such-field
    1) "antirez"
    2) "1977"
    3) (nil)

Присутствует возможность инкремента целочисленных значений в хеш-таблице:

.. code-block:: shell

    > hincrby user:1000 birthyear 10
    (integer) 1987

Множества
^^^^^^^^^

Это классические математические множества:

.. code-block:: shell

    > sadd myset 1 2 3 2
    (integer) 3
    > smembers myset
    1. 3
    2. 1
    3. 2

Есть возможность проверки множества на наличие в нём элемента:

.. code-block:: shell

    > sismember myset 3
    (integer) 1
    > sismember myset 30
    (integer) 0

Для данных этого типа доступны операции над множествами: пересечение, объединение, разность и т.п.
Для получения элемента служит команда SPOP.

Упорядоченные множества
^^^^^^^^^^^^^^^^^^^^^^^

Упорядоченные множества отличаются от классических наличием специального поля (score) для сортировки элементов. Для добавления элементов в упорядоченное множество служит команда ZADD, первым аргументом которой передаётся значение для поля score:

.. code-block:: shell

    > zadd hackers 1940 "Alan Kay"
    (integer) 1
    > zadd hackers 1957 "Sophie Wilson"
    (integer 1)
    > zadd hackers 1953 "Richard Stallman"
    (integer) 1
    > zadd hackers 1949 "Anita Borg"
    (integer) 1
    > zadd hackers 1965 "Yukihiro Matsumoto"
    (integer) 1
    > zadd hackers 1914 "Hedy Lamarr"
    (integer) 1
    > zadd hackers 1916 "Claude Shannon"
    (integer) 1
    > zadd hackers 1969 "Linus Torvalds"
    (integer) 1
    > zadd hackers 1912 "Alan Turing"
    (integer) 1
    > zrange hackers 0 -1 withscores
    1) "Alan Turing"
    2) "1912"
    3) "Hedy Lamarr"
    4) "1914"
    5) "Claude Shannon"
    6) "1916"
    7) "Alan Kay"
    8) "1940"
    9) "Anita Borg"
    10) "1949"
    11) "Richard Stallman"
    12) "1953"
    13) "Sophie Wilson"
    14) "1957"
    15) "Yukihiro Matsumoto"
    16) "1965"
    17) "Linus Torvalds"
    18) "1969"

Присутствуют возможности выборки определённых наборов элементов из упорядоченного множества по полю score (команда ZRANGEBYSCORE) и определение позиции элемента в множестве (команда ZRANK):

.. code-block:: shell

    > zrangebyscore hackers -inf 1950
    1) "Alan Turing"
    2) "Hedy Lamarr"
    3) "Claude Shannon"
    4) "Alan Kay"
    5) "Anita Borg"
    > zrank hackers "Anita Borg"
    (integer) 4

В Redis имеется возможность лексической (по алфавиту) сортировки упорядоченных множеств строковых значений:

.. code-block:: shell

    > zadd hackers 0 "Alan Kay" 0 "Sophie Wilson" 0 "Richard Stallman" 0 "Anita Borg" 0 "Yukihiro Matsumoto" 0 "Hedy Lamarr" 0 "Claude Shannon" 0 "Linus Torvalds" 0 "Alan Turing"
    (integer) 9
    > zrange hackers 0 -1
    1) "Alan Kay"
    2) "Alan Turing"
    3) "Anita Borg"
    4) "Claude Shannon"
    5) "Hedy Lamarr"
    6) "Linus Torvalds"
    7) "Richard Stallman"
    8) "Sophie Wilson"
    9) "Yukihiro Matsumoto"

Для таких множеств также присутствует возможность выборки наборов элементов:

.. code-block:: shell

    > zrangebylex hackers [B [P
    1) "Claude Shannon"
    2) "Hedy Lamarr"
    3) "Linus Torvalds"

Битовые массивы и HyperLogLog
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Эти две структуры данных являются очень специфичными. Поэтому отдельного их описания здесь не будет. При необходимости особенности их применения легко можно узнать из официальной документации.
