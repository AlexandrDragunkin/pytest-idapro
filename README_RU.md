|Build Status| |PyPI|

pytest-idapro
=============

Модуль pytest для интерактивного дизассемблера и IDAPython, путем выполнения внутреннего pytest runner внутри IDA или имитации функциональности IDAPython за пределами IDA.

Motivation
----------

По мере увеличения среднего размера плагина IDAPython необходимость правильных объединений становится все более очевидной. Назначение этого подключаемого модуля pytest состоит в том, чтобы облегчить модульное тестирование подключаемых модулей и сценариев IDAPython и уменьшить разрыв между службами Continuous Integration и исполняемым файлом IDA (который мы не можем выполнить в CI).

Basic usage
-----------

Плагин pytest-idapro добавляет новые флаги командной строки в исполняемый файл pytest.
Все флаги pytest-idapro начинаются с `--ida` и существуют в категории "интерактивные средства тестирования дизассемблеров" (interactive disassembler testing facilities).

Основное использование pytest-idapro будет заключаться в использовании `--ida` для выполнения сеанса pyteset внутри IDA и необязательного флага` --ida-file`:

```console
   $ pytest --ida <PATH TO IDA EXEUTABLE> --ida-file <PATH TO IDB OR SUPPORTED FILE> ;
```
Выполнение приведенной выше команды запустит все тесты, обнаруженные pytest в текущем каталоге внутри экземпляра IDA с заданным файлом, поддерживаемым IDA (IDB, EXE, SO и т. д.).

Record and Replay
-----------------

Начиная с версии 0.3.5, pytest-idapro переключился с фокуса макета на фокус записи и воспроизведения, предоставив два новых флага командной строки; `--ida-record` будет записывать все вызовы API IDAPython и поведение IDA при выполнении тестов в указанный файл json. Используя `--ida-replay` и файл записи json, pytest-idapro может затем воспроизвести среду IDA и поведение API IDAPython без исполняемого файла IDA или флага --ida.

Fixtures
--------

Pytest `Fixtures <https://docs.pytest.org/en/latest/fixture.html>`_ are
exteremly powerful when writing tests, and pytest-idapro currently comes with
two helpful fixtures:

1. `idapro_plugin_entry` - pytest-idapro will automatically identify all
   ida plugin entry points (functions named :code:`PLUGIN_ENTRY`) across your
   code base and let you easily writing tests for all plugin objects defined.
2. `idapro_action_entry` - pytest-idapro will automatically identify all
   ida actions (objects inheriting the :code:`action_handler_t` class)
   throughout your code and again, let you easily write tests for all of your
   actions.

Peeking Under the Hood
======================

By providing the :code:`--ida` flag ipytest-idapro will run a worker pytest
instance inside IDA which will execute all tests, collect results and
communicate with the main pytest process (this behavior is somewhat similar to
the pytest-xdist plugin). By defualt IDA will open a temporary empty database
file unless  the :code:`--ida-file` flag is used to specify IDB or binary file
for IDA to analyze before running any tests.

Recording
---------

In order to record API calls, IDA python objects and their interaction, a
series of proxy/wrapper objects are created for all IDA implemented python
objects (modules, functions, clases, objects, etc). Those proxy objects will
behave identically but will register all interaction between executed code and
the IDAPython interface, which will eventually be dumped to a JSON file.

For the recording to take place as soon as possible, pytest-idapro will modify
IDA's python initialization script (python/init.py). The change is performed
just before starting an IDA instance and revereted as soon as possible.

Instance Matching
-----------------

When an IDA API function is called during replaying, the appropriate return
value must be returned. This is easy when every function is called once, but is
increasingly difficult when the same function is called more then once, and
even more so when it's called more than once with the same arguments.
To correctly return the right value for every call, all calls are recorded with
metadata describing call environment such as the call stack and arguments.
Those are then used to match the correct instance of a call or a class
instantiation while replaying.

Caveats
-------

1. Recording python objects is difficult and requires some object specific
   handling, even more so with the amount of swig, backporting and monkey
   patching in IDA. Therefore, certain APIs may break. Hopefully those will be
   reported and fixed.
2. As mentioned before IDA's init.py file is patched in order to inject the
   code proxy/record system. A special effort was made to revert the
   modifications immidiately regardless of any errors, but if any unexpected
   behavior is observed when pytest-idapro is unused, you may want to check the
   top of :code`<IDA DIR>/python/init.py`.
3. No effort is corrently made into sanitaizing private information from the
   record JSON file. As API results and executable file paths are recorded into
   the json file, details such as paths (and therefore usernames) and other
   personal data may be exposed through the record JSON file. Only use with
   IDBs you don't mind sharing!
4. Instance matching hueristics have their limits, although some types of
   changes won't interfere with the hueristics so much, however ability to
   match old recordings will decrease as more changes are made.
   Users are therefore encouraged to report significantly degrading changes (so
   hueristics will be adjusted accordingly) as well as execute against a real
   IDA instance every once in a while.

.. |Build Status| image:: https://travis-ci.org/nirizr/pytest-idapro.svg?branch=master
   :alt: Build Status
   :target: https://travis-ci.org/nirizr/pytest-idapro
.. |PyPI| image:: https://img.shields.io/pypi/v/pytest-idapro.svg
   :alt: PyPI
   :target: https://pypi.python.org/pypi/pytest-idapro