title: Клаудфлайр как 4to6 прокся
summary: Мотивы и технология переползания на IP6, cloudflare как 4to6 proxy и жадный селектел
pub_date: 2013-05-08
tags: [net]

Переход на IP6
===============

Кроме того, что я катаюсь на паромах-самолетах и пишу криптоанархический Voip,
на мне еще висит немного дурной работы по поддержке чужых старых вебхостов.

Вебхосты засунуты за nginx и особого интереса не представляют, но есть одна
деталь, которая называется "дефицит IP4 адресов". Вебхосты живут на
виртуальной машине, которая хостится на `селектеле`_, который ВНЕЗАПНО стал
чаржить по 70 деревянных в месяц за каждый выданный IP4 адрес. Мотивируют.

.. image:: ip4bill.png

Поскольку машина все равно засунута за реверс-прокси `клаудфрайра`_ и больше
ни для чего, кроме наливания трафика из сети клаудфлайра этот адрес не нужен,
родился хитрый план.

.. image:: dropip4.png

От адреса отказаться, аренду не платить, а клаудфлайровский прокси натравить
сразу на IP6 адрес. Вобщемто план сработал, но с некоторыми хаками.

Nginx
-----

Для начала я устроил генеральную репетицию. Настроил клаудфларовскую проксю
гнать трафик на *другой* IP4 адрес (соседнюю виртуальную машину), а уже оттуда
гнать через IP6 на настоящий бекенд. Ниже схема, на которой квадратными
скобками обозначены хосты, а круглыми адреса/интерфейсы.

::

    [CLIENT](IP4) -> (IP4)[CFLARE ](IP4) -> (IP4)[TESTBOX](IP6) -> (IP6)[WWW]

Для этого на тестбоксе пришлось обновить nginx до 1.3 и написать вот такой
конфиг:

::

    server {
        servername blah-blah;
        location / {
          set $addr "[2a00:ab00:103:0f:00:00:XXXX:XXXX]";
          proxy_pass http://$addr;
          proxy_set_header  Host $host;
          proxy_redirect    off;
        }
    }

В предыдущих версиях nginx такой фокус делать `не умел`_ и ругался, что в конфиге
ошибка. Поэтому плюйте в лицо тому, кто говорит, что весь софт поддерживает IP6
уже 20 лет, а все тормозят и почему-то не юзают. Не поддерживает нихуя.

IP6 listen
----------

Дальше пошли пляски уже с тем нжынксом, который собственно обрабатывает трафик и
потом сует в приложение.

Как было настроено раньше:

::

    /etc/nginx/sites-available/default:
    ...
    server {
        listen   80; ## listen for ipv4; this line is default and implied
    }
    ...

    /etc/nginx/sites-available/blahhost.conf 
    server {
        server_name blah-blah;

        location / {
        ...
        }
    }


Казалось бы, добавить строчку `listen   [::]:80 default ipv6only=on` в главном
конфиге (там же где listen 80), а дальше оно само разберется. А нифига подобного.

Во-первых, нужно действительно добавить эту строчку в дефолтном конфиге, после чего добавить
ее укороченную версию в конфиг каждого виртуального хоста. И обычный `listen 80` не забыть везде,
во время транзишина трафик будет идти обоих типов.

::

    /etc/nginx/sites-available/default:
    ...
    server {
        listen   80; ## listen for ipv4; this line is default and implied
        listen   [::]:80 default ipv6only=on; ## listen for ipv6
    }
    ...

    /etc/nginx/sites-available/blahhost.conf 
    server {
        server_name blah-blah;
        listen 80;
        listen   [::]:80;

        location / {
        ...
        }
    }


Если добавить только `[::]:80` в файл виртуального хоста - отвалится IP4, если добавить только в дефолтный - IP6 уйдет
в какой-то девнул, а не туда, где описан соответствующий сервернейм.

Клаудфлайр
----------

Клаудфларовские товарисчи реализовали сомнительную херню, совместив редактор DNS со списком бэкендов.

Вот например:

::

    someserver A 46.182.27.200
    somerserver AAAA 2a00:ab00:103:0f:00:00:XXXX:XXXX

Если просто добавить две такие записи и включить им режим прокси (желтое облачко),
то настоящие адреса в ответе DNS сервера будут подменены на адреса фронтендов клаудфлайра, 
а трафик клиентов, которые пришли на фронтенд по IP4 пойдет на перый адрес, трафик клиентов,
которые пришли по IP6 пойдет на второй. Обычная прокся, никакогой 4to6 или 6to4:

::

    [CLIENT](IP6) -> (IP6)[CFLARE](IP6) -> (IP6)[BACK]
    [CLIENT](IP4) -> (IP4)[CFLARE](IP4) -> (IP4)[BACK]


Дальше начинается магия. Если в настройках клаудфлайровской админки нажать ручку "Enable IPv6 support",
то прокся начнет анонсировать IP6 адреса даже для тех хостов, которым прописан только
IP4 адрес, превращая сервис в 6to4 проксю:

::

    [CLIENT](IP4) -> (IP4)[CFLARE](IP4) -> (IP4)[BACK]
    [CLIENT](IP6) -> (IP6)[CFLARE](IP4) -> (IP4)[BACK]

это сделано на случай, если клиент умный, а сервер тупой. Мне же понадобилась обратная фиговина,
когда клиент тупой, а сервер жадный.

6to4
----

Очевидное решение было такое:

::

    somerserver AAAA 2a00:ab00:103:0f:00:00:XXXX:XXXX

Добавляем хосту только AAAA запись, нажимаем на желтое облачко и радуемся. Потом перестаем радоваться,
потомучто сервис нифига не анонсирует A запись для этого хоста. В итоге клиент приходит, думает что ничего
нет и уходит. Ой.

::

    [CLIENT](IP6) -> (IP6)[CFLARE](IP6) -> (IP6)[BACK]
    [CLIENT](IP4) -x-> 永恒无效



Но есть очень тупой хак. Можно добавить *обе* записи, сохранить настройки, потом зайти еще раз и одну
убрать. Сервер будет продолжать анонсировать A запись, но гнать трафик на IP6 бекенд.

После редактирования какого-то другого куска зоны магия ломаится и надо переколдовывать заново.


Селектел
--------

Следующий камешек полетит в огород селектела за удобство транзишина.

При создании виртуальной машины, она получат диапазон IP6 адресов, 
часть которого похожа на IP4 адрес этой же машины:

::
    
    IP4 46.182.27.206
    IP6 2a00:ab00:100:46:182:27:206:0000/48

Ну типа удобно.

Поставили машину, настроили адрес из этого диапазона, настроили сервисы,
протестили, что эта шайтан-арба работает.

Нажимаем кнопочку "убрать IP4" - хуууяк, IP6 адрес сразу поменялся, потомучто
старый диапазон должен уйти тому, кто получит соответствующий IP4 адрес.

Пишем в техподдержку, что это адок и неудобноже, техподдержка честно объясняет
механику этого процесса. Все тлен, будущего нет.


.. _селектеле: http://selectel.ru/
.. _клаудфрайра: https://www.cloudflare.com/
.. _не умел: http://trac.nginx.org/nginx/ticket/92

Результат
---------

В результате получается вот так:

::

    [CLIENT](IP4) -> (IP4)[CFLARE ](IP6) -> (IP6)[WWW]
    [CLIENT](IP6) -> (IP6)[CFLARE ](IP6) -> (IP6)[WWW]
                           [CLIENT](IP6) -> (IP6)[SSH]


