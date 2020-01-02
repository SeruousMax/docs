# ACTION
Действие, которое представляет собой параметризуемый блок из нескольких выражений верхнего уровня.

[`DEFINE ACTION`](#define-action) объявляет действие, к которому затем можно многократно обращаться с помощью [`DO`](#do).

`DO ACTION` можно использовать в теле условия [`EVALUATE IF`](#evaluate-if) или цикла [`EVALUATE FOR`](#evaluate-for).

## DEFINE ACTION {#define-action}

Объявляет `ACTION`, состоящее из указанных выражений.

**Синтаксис**

1. `DEFINE ACTION` — объявление действия.
1. [Именованное выражение](expressions.md#named-nodes), по которому объявляемое действие доступно далее в запросе.
1. В круглых скобках — список именованных выражений, по которым внутри действия можно обращаться к параметрам.
1. Ключевое слово `AS`.
1. Список выражений верхнего уровня.
1. `END DEFINE` — маркер последнего выражения внутри действия.

## DO {#do}

Выполняет `ACTION` с указанными параметрами. 

**Синтаксис**
1. `DO` — выполнение действия.
1. Именованное выражение, по которому объявлено действие.
1. В круглых скобках — список значений для использования в роли параметров.

`EMPTY_ACTION` — действие, которое ничего не выполняет.

{% note info %}

В больших запросах объявление действий можно выносить в отдельные файлы и подключать их в основной запрос с помощью `EXPORT + IMPORT`. Для объявления работающих с таблицами действий в отдельной библиотеке, в ней обязательно должно присутствовать `USE my_cluster;`, так как их компиляция зависит от типа кластера.

{% endnote %}

**Пример**

```sql
DEFINE ACTION $hello_world($name) AS
    $name = $name ?? "world";
    SELECT "Hello, " || $name || "!";
END DEFINE;

DO EMPTY_ACTION();
DO $hello_world(NULL);
DO $hello_world("John");
```

## Условное и циклическое выполнение действий {#evaluate-if-for}

### EVALUATE IF {#evaluate-if}

Выполняет `ACTION` по заданному условию. 

**Синтаксис**

1. `EVALUATE IF` — объявление условия.
1. Условие.
1. [`DO`](#do) с именем и параметрами действия.
1. (опционально) `ELSE DO`, чтобы задать действие, если условие не выполнено.

**Пример**

```sql
DEFINE ACTION $hello() AS
    SELECT "Hello!";
END DEFINE;

DEFINE ACTION $bye() AS
    SELECT "Bye!";
END DEFINE;

EVALUATE IF RANDOM(0) > 0.5
    DO $hello()
ELSE
    DO $bye();
```

### EVALUATE FOR {#evaluate-for}

Выполняет `ACTION` для каждого элемента в списке. 

**Синтаксис**

1. `EVALUATE FOR` — объявления цикла.
1. [Именованное выражение](expressions.md#named-nodes), в которое будет подставляться каждый очередной элемент списка.
1. Ключевое слово `IN`.
1. Объявленное выше именованное выражение со списком, по которому будет выполняться действие.
1. [`DO`](#do) с именем и параметрами действия. В параметрах можно использовать как текущий элемент из второго пункта, так и любые объявленные выше именованные выражения, включая сам список.
1. (опционально) `ELSE DO`, чтобы задать действие при пустом списке.

**Пример**

```sql
-- скопировать таблицу $input в $count новых таблиц
$count = 3;
$input = "my_input";
$inputs = ListReplicate($input, $count);
$outputs = ListMap(
    ListFromRange(0, $count),
    ($i) -> {
        RETURN "tmp/out_" || CAST($i as String)
    }
);
$pairs = ListZip($inputs, $outputs);

DEFINE ACTION $copy_table($pair) as
   $input = $pair.0;
   $output = $pair.1;
   INSERT INTO $output WITH TRUNCATE
   SELECT * FROM $input;
END DEFINE;

EVALUATE FOR $pair IN $pairs
    DO $copy_table($pair)
ELSE
    DO EMPTY_ACTION(); -- такой ELSE можно было не указывать,
                       -- ничего не делать подразумевается по умолчанию
```

{% note info %}

Стоит учитывать, что `EVALUATE` выполняется до начала работы операции. Также в рамках `EVALUATE` невозможно использование анонимных таблиц.

{% endnote %}