---
title: Неудачный опыт миграции Electron приложения на ECMAScript модули
description:
date: 2021-02-26
tags:
- Electron
layout: layouts/post.njk
---

Работая над своим стартовым шаблоном для Electron приложений я решил полностью отказаться от CommonJS модулей и использовать исключительно ECMAScript модули (далее ES модули или ESM).

Я очень хочу иметь единый стиль кода везде. В моём проекте, как и у многих, непосредственно исходный код написан с использованием ES модулей, а всё остальное (тесты, файлы конфигурации, дополнительные скрипты для сборки) написано с использованием CommonJS модулей. Меня это сильно напрягает и я хочу чтобы всё было в одном стиле -- ESM.

Кратко о модульных системах в NodeJS

Начиная с 13-й версии NodeJS поддерживает две системы модулей:

CommonJS: для подключения модуля используется функция require();

ECMAScript: для подключения модуля используется ключевое слово import или функция import();

Важно знать:

Вы не можете использовать в одном файле и require и import. Либо то, либо другое.

В ES модуле вы можете подключить другой ES или CommonJS модуль.

В CommonJS модуле вы можете подключать исключительно CommonJS модули.

Как NodeJS выбирает систему для конкретного файла

Тут всё просто. Существует два расширения для файлов: .cjs и .mjs которые определяют что это CommonJS или ES модуль соответственно.

Кроме этого, в package.json вы можете добавить свойство type со значением commonjs или module чтобы определить систему модулей по-умолчанию для файлов с расширением .js.

Проблемы Electron

С чего начинается любое Electron приложение? С файла main.js (или background.js) -- точки входа, которая отвечает за запуск, создание окон, проверку обновлений, работу с системными api и так далее. Если сильно упростить, то обычно этот файл имеет такое содержимое:

const { app, BrowserWindow } = require('electron')  app.whenReady().then(() =>   new BrowserWindow().loadFile('index.html') )  app.on('window-all-closed', () => app.quit())

Ничего особенного: подключение одной зависимости и вызов пары функций.

Я хочу использовать по всему проекту ESM, поэтому установил в package.json "type": "module" и переписал main.js:

- const { app, BrowserWindow } = require('electron') + import { app, BrowserWindow } from 'electron'

И моё ещё не написанное приложение уже падает с ошибкой:

Error [ERR_REQUIRE_ESM]: Must use import to load ES Module

Это меня сильно удивило. Я использовал electron v12 у которого под капотом работает NodeJS 14.15. То есть ESM должны поддерживаться и работать.

Файлы проекта подключаются как CommonJS модули

Оказалось проблема в том, как electron, да и многие другие библиотеки, обрабатываются JavaScript файлы вашего проекта. А делают они это очень просто -- через вызов require:

То есть когда вы запускаете

electron /path/to/main.js

Где-то внутри, electron вызывает

require('/path/to/main.js')

Вызывающий файл electron это CommonJS модуль. А наш файл main.js -- ES модуль. CommonJS модули не могут подключать ES модули. Отсюда и ошибка.

И это главная проблема на пути к светлому ESM-будущему.

Обходные пути

Получается я не могу использовать ES модули из-за архитектуры Electron. Это большая проблема, но всё-таки решаемая.

К счастью, в моём проекте уже использовался сборщик. Так что я мог настроить его таким образом, чтобы преобразовывал ESM в CommonJS модули перед запуском Electron. Но чтобы nodeJS правильно определял какой это модуль конечные файлы нужно сохранять с расширением .cjs и это важно.

Проблемы со сборщиками

К моему большому удивлению, это оказалось сложнее чем я предполагал.

Некоторые инструменты не способны сохранять JavaScript в файлы с расширением не .js. Оно и логично, до недавних пор такой нужды не было.

Например используя tsc или esbuild я не могу изменить расширение итоговых файлов, в том случае если собираю не один, а сразу несколько. Все файлы будут сохранены с расширением .js.

Кроме этого многие сборщики/библиотеки думают про JavaScript файлы как про файлы исключительно с расширением .js. И когда приходит файл с расширением .cjs они не знают что с этим делать.

Например Vite, с которым я работаю, имеет возможность настроить шаблон выходных файлов [filename].[hash].cjs. Но до недавних пор он не понимал какой loader использовать для .cjs и падал с ошибкой. Пришлось отправить отдельный PR (который уже принят) чтобы он воспринимал .cjs файлы как JavaScript.

Проблемы с electron-builder

С исходным кодом я разобрался относительно небольшой ценой. Теперь можно писать ES модули, и Vite будет преобразовывать их в CommonJS модули перед запуском Electron.

Теперь, нужно обработанный код запаковать в исполняемое приложение. Для этого используется electron-builder. И тут меня ждали всё те же проблемы:

electron-builder под капотом использует пакет read-config-file.

read-config-file это CommonJS модуль, который использует require('/path/to/config.js') для чтения конфигурации в моём проекте.

CommonJS модуль не может подключать ES модуль и падает с ошибкой.

Это означает, что я никак не смогу написать конфиг electron-builderкак ES модуль.

Я решил пойти не компромисс и оставить конфиг electron-builder как CommonJS модуль с расширением .cjs. Это ставит крест на идее полностью отказаться от CommonJS модулей в проекте.

Но, это не помогло.

Дело в том, что read-config-file просто не воспринимает файлы .cjs как JavaScript:

if (config.endsWith('.js')) {     require(config) } else {     // ... Это что угодно, но не JavaScript }

Я отправил PR для исправления, но на момент публикации его так и не приняли.

Подчеркну electron-builderв принципе не способен обрабатывать JavaScript файл конфигурации в проектах где в package.json указано "type": "module".

Так что как временное решение пришлось переписать конфигурацию с .js на .json, отказавшись от некоторых возможностей.

Проблемы с экосистемой в целом

И подобный подход встречается в очень многих библиотеках, что вызывает массу проблем, если вы хотите писать ES модулями.

Например eslint. Я не могу написать конфиг для eslint как ESM. В документации так прямо и написано.

И всё по той же причине -- где-то под капотом он использует require чтобы загрузить ваш .eslintrc.js. И единственный способ обойти это переименовать ваш файл в .eslintrc.cjs и продолжить использовать синтаксис CommonJS.

А смена расширений снова приводит к некоторым проблемам. Так моя IDE отказывается определять .eslintrc.cjs как конфиг для eslint.

Другой пример -- dotenv. В документации есть целый раздел посвященный ESM, возможным проблемам и вариантам их решения.

Jest имеет экспериментальную, не стабильную и ограниченную поддержку ES модулей.

Больше проблем чем пользы

В какой-то момент я осознал, что желаемого мною результата не добиться. Я то и дело сталкиваюсь с  проблемами, невозможностью использовать тот или иной пакет, вынужден идти на компромиссы и отказываться от каких-то возможностей. И это всё только на этапе подключения файлов. А ведь различий в модульных системах намного больше чем название одной функции. Так что мне пришлось отказаться от этой идеи на ближайшие несколько лет.

