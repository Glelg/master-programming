==================
Master programming
==================

-------------------------------
Лекция №9 (Обзор boost::spirit)
-------------------------------

:Author: Игорь Шаронов
:Date: 2018-11-06

Знакомство с boost::spirit
==========================

Из чего состоит грамматика языка?
---------------------------------

* Связующее звено между представлениями одних и тех же сущностей
* AST --- abstract syntax tree
* `EBNF (Extended Backus-Naur Form) <https://ru.wikipedia.org/wiki/Расширенная_форма_Бэкуса_—_Наура>`_ --- язык описания грамматики
* Рекурсивность, как полнота грамматики
* PEG (Parsing Expression Grammar) --- Domain Specific Embedded Language

`boost::spirit::x3 <https://ciere.com/cppnow15/using_x3.pdf>`_

Концепты spirit-а
-----------------

``x3::rule<ID, Attribute>``

* **Правила** составляются из **парсеров**, которые захватывают **атрибуты**
  * Synthesized attribute --- атрибут по умолчанию
  * Compatible attribute --- совместимый (конвертируемый) атрибут (``std::string`` и ``std::vector<char>``)
* Грамматика --- набор правил и специальных атрибутов для построения AST

.. figure:: rule.dot.svg

    Пример вывода правил

Примитивные парсеры
-------------------

* ``int_``, ``short_``, ``long_``, ``int_(42)``, ... --- парсеры целых чисел
* ``double_``, ``float_``, ``double_(42.2)``, ... --- парсеры действительных чисел
* ``bool_``, ``true_``, ``false_`` --- булевы парсеры
* ``lit("abc")``, ``char_``, ``char_("A-Za-z")``, ... --- литеральные парсеры (точное соответствие строке)
* ``alnum``, ``blank``, ``space``, ``lower``, ... --- классификаторы
* ``parse`` --- разбор выражения
* ``phrase_parse`` --- более "тонкий" разбор выражения

.. code:: cpp

    namespace x3 = boost::spirit::x3;
    std::string_view text{"123546"};
    bool parsed = x3::parser(text.begin(), text.end(), x3::int_);

Основные операторы boost::spirit
--------------------------------

.. list-table:: PEG Operators
    :header-rows: 1

    * - Description
      - PEG
      - Spirit X3
      - Example
    * - Sequence
      - a b
      - a >> b
      - ``int_ >> ' ' >> double_``
    * - Alternative
      - a|b
      - a|b
      - ``alpha | digit``
    * - Zero or more (Kleene)
      - a*
      - \*a
      - ``alpha >> *(',' >> alnum)``
    * - One or more (Plus)
      - a+
      - +a
      - ``+alnum >> '_'``
    * - Optional
      - a?
      - -a
      - ``-alpha >> int_``
    * - And-predicate
      - &a
      - &a
      - ``int_ >> &char_(';')``
    * - Not-predicate
      - !a
      - !a
      - ``~char_('"')``
    * - Difference
      -
      - a - b
      - ``"/*" >> *(char_ - "*/") >> "*/"``
    * - Expectation
      -
      - a > b
      - ``lit('c') > "++"``
    * - List
      -
      - a % b
      - ``int_ % ','``

Формирование правил
-------------------

* Именованные парсеры
* Можно указать тип атрибута
* Позволяет использовать рекурсивность парсеров
* Предоставляет обработчик ошибок (``on_error``)
* Использование пользовательских функция для определения успешного парсинга (``on_success``)

Примеры парсеров
================

Последовательный парсинг
------------------------

.. code:: cpp

    namespace x3 = boost::spirit::x3;
    std::string_view text{"123 45.6"};
    bool parsed = x3::parser(text.begin(), text.end(), x3::int_ >> ' ' >> x3::double_);

Парсер файла формата ключ-значение
----------------------------------

.. code:: cpp

    std::string_view text{R"(foo: bar,
        gorp : smart ,
        falcou : "crazy frenchman",
        name:sam)"};

    x3::phrase_parse(text.begin(), text.end(),
                     (name >> ':' >> (quote | name)) % ',',
                     x3::space);

Правила
-------

.. code:: cpp

    auto name = x3::rule<class name, std::string>{} // явное прописывание тэга и атрибута
              = x3::alpha >> *x3::alnum; // переменные в C-стиле
    auto quote = '"' >> x3::lexeme[*(~x3::char_('"'))] >> '"'; // строковые значения в кавычках

Атрибуты
========

Получение результата парсинга
-----------------------------

* AST --- удобное представление через ассоциативного рекурсивного массива
* Атрибут --- результат, который предоставляет конкретный парсер
* Литералы не имеют атрибутов
* Примитивные парсеры (``int_``, ``double_``, ...) имеют примитивные по типу атрибуты (``int``, ``double``, ...)
* Нетерминалы вида ``x3::rule<ID, A>`` имеют атрибут ``A``


.. list-table:: Атрибуты операторов
    :header-rows: 1

    * - Оператор
      - Его синтезируемый атрибут
    * - ``a >> b``
      - ``tuple<A, B>``
    * - ``a | b``
      - ``boost::variant<A, B>``
    * - ``*a``, ``+a``
      - ``std::vector<A>``
    * - ``-a``
      - ``boost::optional<A>``
    * - ``&a``, ``!a``
      - нет атрибута
    * - ``a % b``
      - ``std::vector<A>``

Примеры
-------

.. code:: cpp

    std::string_view text{"pizza"};

    std::string result; // совместимо с атрибутом std::vector<char>
    x3::parse(text.begin(), text.end(), *x3::char_, result);

.. code:: cpp

    std::string_view text{"cosmic pizza"};

    std::string result0;
    std::string result1;
    x3::parse(text.begin(), text.end(), *(~x3::char_(' ')) >> ' ' >> *x3::char_, result0, result1);

    std::pair<std::string, std::string> result;
    x3::parse(text.begin(), text.end(), *(~x3::char_(' ')) >> ' ' >> *x3::char_, result);

Разбор с атрибутами примера ключ-значение
-----------------------------------------

.. code:: cpp

    std::string_view text{R"(foo: bar,
        gorp : smart ,
        falcou : "crazy frenchman",
        name:sam)"};

    auto name = x3::alpha >> *x3::alnum;
    auto qoute = '"' >> x3::lexeme[*(~x3::char_('"'))] >> '"';
    auto item = x3::rule<class item, std::pair<std::string, std::string>>>{}
              = name >> ':' >> (quote | name);

    std::map<std::string, std::string> dict;
    x3::phrase_parse(text.begin(), text.end(), item % ',', x3::space, dict);

::

    a: char, b: vector<char> → (a >> b): tuple<char, vector<char>> → vector<char> → string
    a: unused, b: vector<char>, c: unused → (a >> b >> c): vector<char> → string
    a: string, b: string → (a | b): variant<string, string> → string
    a: string, b: unused, c: string → (a >> b >> c): tuple<string, string>
    a: pair<string, string>, b: unused → (a % b): vector<pair<string, string>> → map<string, string>

Построение грамматик
====================

Что нужно для полного построения правила?
-----------------------------------------

#. Тип атрибута
#. Идентификатор правила
#. Тип правила
#. Определение правила
#. Само правило

.. code:: cpp

    struct my_type { ... };
    struct my_rule_class;
    const x3::rule<my_rule_class, my_type> my_rule_type = "my_rule";
    const auto my_rule_def = x3::lexeme[(x3::alpha | '_') >> *(x3::alnum | '_')];
    BOOST_SPIRIT_DEFINE(my_rule)

Обработка ошибок
----------------

https://www.boost.org/doc/libs/1_68_0/libs/spirit/doc/x3/html/spirit_x3/tutorials/error_handling.html

https://github.com/djowel/spirit_x3/blob/master/example/x3/calc5.cpp

Что дальше
----------

* Рекурсивные типы через ``boost::forward_ast``
* Оператор строгого следования ``>``
* Явное прописывание генерируемого атрибута: ``"abc" > x3::attr(10)``


.. figure:: https://www.boost.org/doc/libs/1_68_0/libs/spirit/doc/html/images/spiritstructure.png

    Структура ``boost::spirit``

.. figure:: https://www.boost.org/doc/libs/1_68_0/libs/spirit/doc/html/images/spiritkarmaflow.png

    Разница между qi и karma

* И где здесь ``x3``??
