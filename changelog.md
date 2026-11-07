# Changelog

Fork of [meshcentral-community/MeshCentral-FileDistribution](https://github.com/meshcentral-community/MeshCentral-FileDistribution),
created for personal use.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

### Added
- **Plugin permissions in the MeshCentral permissions dialog.** The plugin now registers its permissions in the plugin permissions panel (MeshCentral 1.1.58+), and full site administrators always pass every check.

## [0.16.0]

### Added
- **Progress bar directly in the device label.** While a file is being sent to a device, the label fills green from left to right and shows the percentage. When the file is delivered, the label becomes fully green — the same look as before.
- In the collapsed distribution row, it is now visible how many devices are currently receiving the file and the average percentage, without expanding the list.
- While a file is actually being transferred, the overview refreshes every 1.2 seconds instead of five, so the progress bar moves smoothly. Once the transfer ends, the refresh rate returns to normal.

### How this is calculated
The progress is taken from the server, not from the device: the server tracks the transfer itself and knows how many bytes it has sent. Because of that, the agent module did not change and no kernel update is required. The value is stored only in server memory, not in the database, and disappears as soon as the transfer ends.

The bar shows what the server has sent, while delivery confirmation still comes from the device after it verifies the file size. So 100% on the bar means “the server sent everything,” while the green label without a bar means “the file is fully present on disk.”

## [0.15.1]

### Fixed
- Rows in the “Current distributions” list did not expand. A distribution is identified by a key built from two paths joined by a null byte, and that key was passed through a markup attribute — but a null byte is not preserved in that transfer, so the value never matched the original. The attribute now carries the row number, and the key is read from the already built list of distributions.

## [0.15.0]

### Added
- The “Current distributions” section now adjusts its height to fit the content by default. Once it is dragged by the corner, the height becomes fixed and is remembered.
- Each distribution is collapsed into a single row. In the “Devices” column, you can see how many of how many were delivered, and if there are errors, how many were not delivered. It expands by clicking the triangle or the same cell; expanded rows are remembered in the browser and survive auto-refresh.
- In the “My Files” tree, each file shows whether it is being distributed. For folders, it shows how many files inside participate in distributions.

### Note
The flag is updated together with the distribution overview, including automatically while files are still in the path. The selected files and expanded folders are preserved.

## [0.14.2]

### Fixed
- **Delivery states were not being saved, and the server log showed `Update document requires atomic operators`.** The state write went directly to the MongoDB driver, bypassing the wrapper that wraps fields in `$set`. MongoDB rejects this update. The call now goes through the same wrapper used by the rest of the plugin.
- State writing is also protected by a try/catch: the driver can report an error by throwing instead of returning a rejected promise, and in that case the normal handling never runs. A state write failure can no longer break the plugin.

## [0.14.1]

### Changed
- Expanded folders are remembered in the browser and restored after distribution and after page reload.
- A folder is identified by its path instead of its index. The index shifted when files were added or removed in “My Files,” causing the wrong folders to expand after those changes.

## [0.14.0]

### Added
- **Files from “My Files” are shown as a folder tree**, rather than a flat list of full paths. Next to each folder name, the number of files in it, including nested ones, is shown.
- Selecting a folder selects all files inside it, including nested folders at any depth. Clearing the selection removes only those files.
- The folder checkbox reflects the contents: checked when everything inside is selected, and shown as intermediate when only part is selected.
- Links for “Expand all” and “Collapse all.” By default, folders are collapsed so a long list does not take over the whole block.

### Fixed
- The literal `\n` appeared under the file list — a leftover from an incorrectly inserted markup string.

## [0.13.0]

### Added
- **In “Current distributions,” it is visible whether each file was delivered to each device.** The device label is colored: green — the file was delivered and confirmed by the device itself, blue — currently being transferred, gray — waiting to be sent, red — not delivered. Hovering shows an explanation and details.
- Next to the number of devices, it now shows how many of them already received the file: “delivered 13 of 16.” This makes it easier to judge when the next batch can be started.
- While something is in the path, the overview refreshes automatically every five seconds and stops once all devices have confirmed delivery.

### How this is determined
- Confirmation comes from the agent, not the server: the device reports success only after verifying the size of the received file against the expected value. So green means “the file is fully on disk,” not “the server finished sending it.”
- A periodic check every 20 minutes confirms the state again, so distributions created before this version will gradually be colored as well, and a file deleted manually on the device will stop being counted as delivered.

## [0.12.0]

### Fixed — important
- **The MeshCentral server could be killed by the system due to memory exhaustion while distributing large files to multiple machines.** The transfer inherited from the original project read the file as a stream and pushed each chunk directly into the agent socket without any control. The disk is fast, the network is slow, so all unsent data accumulated in memory, and hex encoding doubled the size: 61 MB to ten machines meant well over a gigabyte queued at once.
- Transfers are now queued, only two run at a time, and the next chunk is read only after the socket has drained. If the MeshCentral version does not report socket depth, a fixed pace is used; the amount of memory used is limited in both cases.
- Transfer stops immediately when the agent disconnects, and the queue continues to work.
- A repeated request for the same file to the same device is ignored while the transfer is ongoing or waiting in the queue.

### Measured
On a sample: a 6 MB file sent to ten machines. The old approach kept around 120 MB in memory; the new one keeps about 4.3 MB for the same volume of transferred data. Memory usage is now almost independent of file size and the number of machines.

## [0.11.0]

### Added
- **The destination directory is created on the device if it does not exist.** Previously, sending to a non-existent folder simply failed: the write crashed, and the log only showed `write failed`.
- The entire missing hierarchy is created, not just one level: for `C:\install\deep\nested`, all three folders are created.
- The directory is prepared ahead of time when the distribution is received, rather than at the moment the first chunk is written. This matters when the agent does not have `mkdirSync` and must call a system command instead: it is asynchronous, and the extra time comes from fetching the file from the server.
- Right before the actual write, the directory presence is checked again. If it is still missing, the transfer stops with a clear reason in the report instead of a write error.

### Notes
- Network paths are handled correctly: `\\server\share` is left alone, and only nested folders inside the share are created.
- A failure to gain permission to create the folder does not crash the module: the reason is written to the log, and a fallback attempt is made with a system command.

## [0.10.7]

### Fixed
- **The button on “My Devices” disappeared after a normal page refresh.** The actual cause was different from what was assumed in 0.10.6. On startup, the plugin asked the server whether to show the button and inserted it only after receiving the answer. During a page refresh, the request could be sent before the connection to the server was ready, and the answer never arrived, so the button never appeared. “Save” helped because by then the connection was already established.
- The last known setting is now stored in the browser, and the button appears immediately on load without waiting for the server — the same way Quick Commands does it. Requests to the server are retried until a response arrives, and if needed the response corrects the button state.

### About the fix in 0.10.6
The assumption that MeshCentral rebuilds the toolbar was disproved by the Quick Commands code: it inserts the button once and never restores it, yet it still does not disappear. The observer from 0.10.6 remains as a safeguard: it does not interfere and costs only one ID lookup per batch of changes.

## [0.10.6]

### Fixed
- **The distribution button disappeared from the “My Devices” panel.** MeshCentral rebuilds the toolbar when the view, filter, or page changes, and the button was destroyed along with the old markup. The insertion happened only once and was marked with a flag, so the button did not reappear until the page was reloaded. “Save” in the settings only helped because users refreshed the page after that.
  The insertion now checks whether the button is actually present on the page, and a markup observer restores it after each toolbar rebuild. The markup change checks are grouped into a single pass to avoid overloading the page.
- Turning the button off in the settings now removes it immediately, without reloading.
- The “enabled / disabled” state is now tracked against the current group-action button instead of the one that existed before the toolbar was rebuilt.

## [0.10.5]

### Fixed
- **Unpacking did not start even though the flag was received and the file arrived intact.** The call was deferred via `setTimeout`, and in the agent environment a timer that nobody keeps a reference to can be garbage-collected before firing — so no log entry ever appeared. It is telling that the original code stores its periodic timer in a module variable for exactly this reason. The deferred call was removed: after the process launch fix in 0.10.1 it was no longer needed, the file was already written and the stream was closed, and the unpacking procedure itself is protected by its own error handling.
- In `filedist.txt`, it is now clear exactly what was saved for each distribution (`stored map ... unzip=true`) and which flag value passed the check before unpacking.

## [0.10.4]

### Исправлено — важно
- **Убрано расширение `Array.prototype.remove`.** Оно присутствовало в
  оригинале плагина с 2020 года, но не использовалось ни в нём, ни здесь.
  Расширение прототипа видно всей JS-среде агента, поэтому любой посторонний
  код, перебирающий массив через `for-in`, получал лишний ключ `remove`. Это
  вероятная причина отказов штатной распаковки в файловом менеджере
  MeshCentral и сопутствующих исключений с промисами.
- При загрузке модуль пишет в `filedist.txt` строку вида
  `filedist module 0.10.4 loaded`. По ней сразу видно, обновилось ли ядро
  агента, — раньше это приходилось выяснять косвенно.

### Как проверить
1. После обновления ядер в `filedist.txt` должна появиться строка о загрузке
   модуля с номером версии. Если её нет, ядро на устройстве старое, и всё
   остальное проверять рано.
2. Если штатная распаковка в файловом менеджере заработает, причина
   подтвердится. Если нет — отключите плагин целиком и проверьте снова: так
   станет ясно, связано ли это с ним вообще.

## [0.10.3]

### Исправлено
- **Распаковка не запускалась, хотя флаг доходил до устройства.** Агент при
  получении карты собирал новый объект только из пути и размера, отбрасывая
  всё остальное, — этот код достался от оригинала. Флаг до хранилища не
  долетал, и проверка перед распаковкой всегда видела «нет». Исправлено в
  обеих точках: и при добавлении одной раздачи, и при получении полного
  набора карт.
- Каждый ранний выход из процедуры распаковки теперь оставляет запись в
  `filedist.txt`: запрошена ли распаковка вообще, и если она не состоялась —
  по какой причине. Раньше часть отказов проходила молча.

## [0.10.2]

### Добавлено
- Чекбокс распаковки появился и на вкладке File Distribution самого устройства.
  Раньше он был только на странице массовой раздачи, поэтому при раздаче с
  вкладки устройства архив всегда приходил без распаковки.

### Диагностика
- Сервер пишет в отладочный лог, с каким значением флага распаковки пришёл
  запрос: строка `addFileMap ... unzip=true|false` для одиночной раздачи и
  `bulk add: N file(s), unzip flags: 0101` для массовой. Это позволяет сразу
  отличить непоставленную галочку от потери флага по дороге.

## [0.10.1]

### Исправлено — важно
- **Агент падал с uncaughtException при распаковке, и файл переставал
  скачиваться.** Дочерний процесс запускался без подписки на потоки вывода и
  без удержания ссылки на объект. Среда агента этого не допускает: возникает
  необработанное исключение, которое утаскивает за собой и вызвавший код.
  Запуск процессов сведён в одну функцию, которая подписывается на `stdout` и
  `stderr`, держит ссылку до завершения и сообщает код возврата.
- Распаковка вынесена из обработчика передачи и запускается отдельно, после
  того как обработчик завершился. Теперь её сбой не может повлиять на запись
  файла — а именно это и произошло в 0.10.0.
- Запасной путь удаления через системную команду переведён на ту же функцию.
  Он существовал с 0.5.1, но ни разу не выполнялся, поэтому ошибка оставалась
  незамеченной до первой настоящей распаковки.

### Примечание
Если распаковка не заработает и на этой версии, раздача файлов продолжит
работать: достаточно не ставить галочку. Причина будет видна в отчёте агента
в отладочном логе сервера и в `filedist.txt`.

## [0.10.0]

### Добавлено
- Распаковка распространена на `.7z` и `.rar`. Если на устройстве установлен
  7-Zip, агент использует его — он же становится предпочтительным и для `.zip`.
  Ищется он по стандартным путям установки.
- Без 7-Zip поведение прежнее: `.zip` разворачивается через `Expand-Archive`,
  а `.7z` и `.rar` не трогаются, и причина попадает в отчёт агента.
- В окне подтверждения раздачи появляется предупреждение, когда в выборе есть
  форматы кроме `.zip`.

### Почему не внутренний модуль агента
Консольная команда `unzip` в агенте построена на zip-ридере: подтвердить его
программный интерфейс изнутри модуля не удалось, и для `.7z` и `.rar` он всё
равно бесполезен. Внешний 7-Zip покрывает все три формата и проверяем на месте.

## [0.9.0]

### Добавлено
- Выбор файлов переведён на чекбоксы вместо выделения кликом.
- Необязательная распаковка архива на устройстве после раздачи. Чекбокс
  находится в окне подтверждения раздачи и доступен, только когда все
  выбранные файлы — архивы `.zip`.
- Архив распаковывается в папку рядом с ним, названную по имени архива без
  расширения: `C:\install\app.zip` разворачивается в `C:\install\app`.
  Сам архив остаётся на месте.

### Особенности реализации
- Распаковка выполняется через `Expand-Archive` в PowerShell, а не через
  внутренний zip-модуль агента: его программный интерфейс подтвердить не
  удалось, а `Expand-Archive` есть на всех поддерживаемых версиях Windows и
  ведёт себя предсказуемо. Отсюда два ограничения: только Windows и только
  `.zip`.
- Распаковка запускается лишь после того, как сверка размера подтвердила, что
  файл пришёл целиком. Повреждённый архив не разворачивается.
- Результат сообщается серверу и виден в отладочном логе.
- Флаг хранится в записи раздачи, поэтому при переподключении устройства и при
  обновлении файла на сервере поведение сохраняется.

### Известные ограничения
- Снятие раздачи не удаляет распакованную папку: её содержимое плагину
  неизвестно, и удалять его вслепую опасно. Папку нужно убирать вручную.
- Распаковка идёт поверх существующих файлов (`-Force`).

## [0.8.0]

### Добавлено
- Множественный выбор файлов в блоке «Файл из „Мои файлы"». Клик по файлу
  добавляет или убирает его из выбора; рядом со списком показано число
  выбранных файлов и ссылка «Снять выбор».
- Все выбранные файлы раздаются в одну общую папку, каждый под своим исходным
  именем. Предпросмотр показывает итоговые пути построчно (первые восемь).
- Поле «Сохранить как» отключается, когда выбрано больше одного файла: имя
  берётся у каждого файла своё, и переименование теряет смысл.
- В итоговом сообщении при нескольких файлах указывается и число файлов, и
  число созданных записей на устройствах.

### Особенности реализации
- Сервер принимает список файлов одной операцией. Каждый файл проходит через
  ту же проверенную процедуру, что и раньше, поэтому проверки прав, отсев
  дубликатов и пачечная обработка работают без изменений.
- Права на устройство проверяются один раз на всю операцию, а не для каждого
  файла: при трёх файлах на 50 устройств это 50 обращений к базе вместо 150.
- Ограничение — 5000 записей за операцию, то есть, например, 20 файлов на 250
  устройств. Превышение отклоняется с понятным сообщением.
- Старый формат запроса с одним файлом сервер по-прежнему принимает.

## [0.7.3]

### Исправлено
- Не переводились подпись «Будет записан в…», заголовки колонок в «Текущих
  раздачах», кнопка снятия раздачи со всех устройств и сообщение о пустом
  списке. Переводы в словаре были, но два элемента не были размечены ключами,
  у третьего изменилась разметка после того, как ключи расставлялись, а
  четвёртый рисуется кодом и требовал вызова функции перевода.
- Проведена сплошная проверка обоих шаблонов: непереведённых строк не осталось
  ни в разметке, ни в коде.

### Изменено
- Описание плагина в `config.json` переведено на русский.

## [0.7.2]

### Исправлено
- Переведены тексты, оставшиеся английскими в версии 0.7.0: содержимое диалогов
  раздачи и снятия раздачи, подписи галочек удаления файла, пояснения под ними,
  заголовок диалога на вкладке устройства и уведомление о количестве устройств,
  перенесённых из «Мои устройства».
- В словарь добавлены недостающие ключи; все строки, используемые в коде,
  теперь имеют перевод. Английским остаётся только сообщение о недоступном
  окне MeshCentral — оно показывается до загрузки словаря и служит запасным
  вариантом.

## [0.7.1]

### Документация
- В readme добавлен раздел о происхождении: что взято как код (оригинал Ryan
  Blenis под Apache-2.0, форк сообщества, сама MeshCentral) и что изучалось
  как подход без заимствования кода (QuickCommands, форк ScriptTask от
  InnovoDeveloper).
- Описаны намерения: проект сделан для одного сервера, репозиторий открыт лишь
  потому, что MeshCentral ставит плагины по публичной ссылке, продуктом не
  является и аудиторию оригинала не оспаривает.
- Добавлен раздел о языке интерфейса и о том, как добавить новый.

## [0.7.0]

### Добавлено
- Локализация интерфейса: русский и английский. Язык определяется
  автоматически — по языку MeshCentral, затем по языку браузера — и
  переключается вручную в настройках плагина. Выбор хранится в браузере, у
  каждого пользователя свой.
- Строки вынесены в `includes/fdlang.js`, общий для страницы управления и
  вкладки устройства. Английский служит источником и запасным вариантом: ключ
  без перевода показывается по-английски, а не в виде имени ключа, поэтому
  неполный словарь ничего не ломает.
- Добавление нового языка сводится к копированию блока `en` в `fdlang.js`;
  список в настройках строится по словарю сам.

## [0.6.1]

### Исправлено — важно
- **Двоичные файлы приходили повреждёнными и увеличенными.** Причина внесена в
  версии 0.5.0: флаг открытия файла на агенте был изменён с `wb` на `w`. Для
  Node буква `b` действительно не значит ничего, но файловый модуль агента её
  учитывает, и без неё запись идёт в текстовом режиме — каждый байт `0x0A`
  превращается в `0x0D 0x0A`. На файле 3,7 МБ это дало 13919 лишних байт и
  неработающий исполняемый файл. Флаг возвращён к `wb`.
- Ограничение на одновременные передачи и сверка размера, добавленные в 0.6.0,
  сохранены: первое исключает наложение двух передач, второе не даёт остаться
  на диске файлу неверного размера.

### Проверка
После обновления ядер снимите и создайте раздачу заново. В `filedist.txt`
агента должна появиться строка `transfer of ... complete, N bytes`, где N равно
исходному размеру файла.

## [0.6.0]

### Исправлено — важно
- **Файл приходил на устройство повреждённым и большего размера.** Поток записи
  открывался, только если ещё не открыт, и закрывался по маркеру конца передачи.
  Запрос файла при этом отправляется из двух мест — при получении карты и при
  периодической проверке, — поэтому второй запрос мог начаться до завершения
  первого, и данные обеих передач писались в один поток, дописываясь друг к
  другу. Оборвавшаяся передача оставляла поток открытым навсегда, и всё
  последующее приписывалось к нему.
  Теперь одновременно идёт не более одной передачи файла: повторный запрос
  отклоняется, а передача, не завершившаяся за 10 минут, считается мёртвой и
  начинается заново с усечением файла.
- По окончании передачи агент сверяет размер полученного файла с ожидаемым.
  Неполный или лишний файл удаляется, а не остаётся на диске в нерабочем виде;
  сервер получает отчёт с обоими размерами.
- Отказ удаления по несовпадению размера, замеченный ранее, был следствием этой
  же ошибки: защита верно определяла, что файл на диске отличается от
  розданного. Отдельной правки она не требует.

### Примечание
Файлы, розданные предыдущими версиями, могли быть повреждены. Проверьте размер
на устройствах: раздачи, где он отличается от исходного, нужно снять и создать
заново.

## [0.5.3]

### Исправлено
- Список файлов в панели раздачи прижимался к нижнему краю: строка выравнивала
  содержимое по базовой линии, что верно для коротких строк, но не для той,
  которая растягивается на всю высоту панели. Список файлов теперь начинается
  на одном уровне с полем «Folder on the devices».

## [0.5.2]

### Исправлено
- Страница, открытая кнопкой с My Devices в новой вкладке, была пустой: она
  читает устройства, файлы и соединение из окна MeshCentral через `parent`, а в
  отдельной вкладке `parent` — она сама. Теперь окно-хозяин определяется как
  `parent` внутри фрейма и `opener` в отдельной вкладке; если это окно закрыто,
  выводится понятное объяснение вместо пустой страницы.
- «Distribute a file to many devices» и «Devices» разделены на самостоятельные
  панели с независимой регулировкой высоты.

### Диагностика
- Агент отчитывается серверу о каждом снятии раздачи, в том числе когда файл
  сохраняется, и указывает версию своего модуля. В отладочном логе MeshCentral
  видно, дошла ли команда и какой именно ветвью она завершилась; пометка
  `agent module pre-0.5.2` означает, что на устройстве старое ядро.

## [0.5.1]

### Исправлено
- Файл не удалялся с устройства ни точечно, ни при снятии раздачи со всех.
  Удаление опиралось на `fs.unlinkSync`, которого в урезанном файловом модуле
  агента может не быть, и молча выходило. Теперь по очереди пробуются
  `unlinkSync`, `unlink` и системная команда удаления, результат проверяется
  повторным `stat`, а агент отчитывается серверу — отказ виден в отладочном
  логе MeshCentral без включения отладки внутри агента.
- Сверка размера перед удалением больше не отказывает, если агент потерял свою
  запись о раздаче (перезапуск, обновление ядра): размер теперь приходит с
  сервера вместе с командой. Сама защита сохранена — изменённый на устройстве
  файл по-прежнему не удаляется.
- Кнопка на My Devices не появлялась: настройка приходит с сервера асинхронно,
  и к этому моменту панели инструментов может ещё не быть. Вставка теперь
  повторяется до 20 раз и заново выполняется при смене страницы через
  официальный хук `goPageStart`.

### Изменено
- Регулируется высота самих разделов, а не блоков внутри них: дочерние списки
  растягиваются вслед за разделом. Высоты запоминаются в браузере.
- Раздел «Current distributions» можно сжать значительно сильнее прежнего,
  список устройств внутри прокручивается.

## [0.5.0]

### Исправлено
- Кнопки All / Online / Offline не переключали список: обработчик `fdSetPill`
  вызывался из разметки, но не был объявлен в коде, поэтому клик молча падал.
- Ответы с настройкой кнопки на My Devices доставлялись не тем механизмом,
  которым в этом плагине доставляются остальные ответы сервера, и приходили в
  другой форме. Переведены на путь, уже подтверждённый работой обзора раздач.

### Добавлено
- Точечное удаление: у каждого устройства в списке раздачи появился крестик,
  снимающий раздачу только с него.
- Три панели (раздача, устройства, обзор) тянутся за правый нижний угол
  независимо друг от друга; высоты запоминаются в браузере.
- Последние 10 введённых папок назначения запоминаются и предлагаются при
  вводе.
- Фильтрация устройств по тегам — отдельной колонкой рядом с группами, по тому
  же принципу «ИЛИ внутри колонки, И между колонками».
- Кнопка «Distribute File» на панели инструментов My Devices, по аналогии с
  Quick Commands: открывает страницу управления с уже выбранными устройствами.
  Включается в настройках страницы, доступных сайт-администратору.

### Изменено
- Флаг открытия файла на агенте изменён с нестандартного `wb` на `w`. Замена
  одноимённого файла и раньше происходила безусловно, но теперь усечение
  старого содержимого гарантировано.

### Известные ограничения
- Кнопка на My Devices встраивается в разметку, которую MeshCentral официально
  для этого не предоставляет. Тот же приём использует Quick Commands, но при
  смене темы или версии она может перестать появляться. На остальную работу
  плагина это не влияет.
- Размещение кнопки на вкладках General / Desktop / Terminal в этой версии не
  делалось: там потребовалась бы вставка в разметку конкретных вкладок, а
  именно на подобных вставках форк уже спотыкался.

## [0.4.1]

### Исправлено
- Страница в My Server → Plugins отдавала `Unauthorized`. MeshCentral открывает
  её без каких-либо параметров запроса, а код ожидал `admin=1` и проваливался
  в отказ. Страница управления теперь отдаётся по умолчанию.

## [0.4.0]

### Добавлено
- Страница управления в **My Server → Plugins**: массовая раздача файла на
  выбранные устройства и обзор всех существующих раздач.
- Выбор устройств с фильтрами: все / онлайн / офлайн со счётчиками, отбор по
  группам устройств, поиск по имени, кнопки «выбрать показанные» и «снять
  показанные», счётчик выбранного.
- Массовое удаление раздачи сразу со всех устройств, с той же галочкой удаления
  файла и той же проверкой размера, что и при точечном удалении.
- Кнопка разворота панели. Где браузер разрешает полноэкранный режим,
  используется он; иначе просто увеличиваются списки.

### Особенности реализации
- Список устройств берётся из интерфейса, но авторитетом остаётся сервер: права
  на каждый узел проверяются заново перед записью, недоступные узлы молча
  пропускаются. Подделка списка в браузере ничего не даёт.
- Массовые операции идут пачками по 10 узлов с паузой, чтобы не блокировать
  сервер. На 227 узлах проход занимает около полусекунды.
- Повторная раздача того же файла в тот же путь не создаёт дубликат.
- Ограничение — 1000 устройств за одну операцию.

## [0.3.0]

### Изменено
- При добавлении раздачи теперь указывается только папка на устройстве, а имя
  файла подставляется из исходного файла в My Files. Поле «Save as» позволяет
  сохранить под другим именем, если нужно. Итоговый путь показывается тут же,
  до нажатия Add.
- Разделитель пути определяется по введённой папке: обратный слэш для путей
  вида `C:\...` и UNC, прямой — для остальных. Лишние слэши в конце папки
  отбрасываются.

Формат хранения не изменился: в базе по-прежнему лежит полный путь, поэтому
существующие раздачи продолжают работать.

## [0.2.1]

### Безопасность
- Перед удалением файла агент сверяет его размер с тем, который был записан
  при раздаче. Если файл на устройстве заменили или изменили, он остаётся
  нетронутым, а причина отказа пишется в отладочный лог. Удаление также
  отклоняется, если агент не знает эту карту или размер не был записан.

## [0.2.0]

### Добавлено
- При удалении раздачи можно выбрать, удалить ли файл с устройства.
  Без галочки файл остаётся на месте и просто перестаёт обновляться —
  это прежнее поведение. С галочкой агент удаляет файл.
  Для удаления устройство должно быть онлайн.

### Исправлено
- Агент не очищал список карт при получении полного набора от сервера,
  хотя в отладку писал «resetting maps». Удалённые раздачи оставались в
  памяти агента и проверялись каждые 20 минут до его перезапуска.

## [0.1.0] - форк

### Исправлено
- Кнопка `[+]` не работала в современной теме. Плагин рисовал диалоги через
  `parent.setDialogMode()`: в `default3.handlebars` эта функция существует и
  вызывается без ошибок, но окно не показывает. Плагин больше её не использует
  и рисует свои модальные окна внутри собственного iframe.
- Из iframe был недоступен `parent.currentNode` — в современной теме он не
  находится в области видимости `window`. Из-за этого падало добавление карты
  и не обновлялся список. Идентификатор устройства теперь берётся из параметра
  `node` в адресе iframe.
- Двойной вызов колбэка в `nemongo.js` при работе с NeDB.
- Неверная проверка администратора в `handleAdminReq`
  (`(user.siteadmin & 0xFFFFFFFF) == 1` вместо сравнения с `0xFFFFFFFF`).
- `checkFileSizes` падал на записи с непригодным путём вместо того, чтобы её
  пропустить.
- `clearTimeout` заменён на `clearInterval` для интервального таймера.
- Убран вызов `parent.pluginHandler.scripttask.resizeContent()` — копипаста из
  другого плагина, ломавшая интерфейс там, где ScriptTask не установлен.

### Безопасность
- `serveraction` не проверял права вообще: любой аутентифицированный
  пользователь мог отправить `addFileMap` и раздать произвольный файл с
  сервера на произвольный узел, включая невидимые ему. Теперь каждое действие
  проверяет права вызывающего на конкретный узел.
- `deleteMap` берёт узел из сохранённой записи, а не из сообщения, поэтому
  нельзя удалить чужую карту, указав свой узел.
- Запрос списка карт (`?user=1&node=...`) требует прав на этот узел.
- Пути к файлам на сервере проверяются на выход за пределы папки файлов, а
  итоговый путь дополнительно сверяется с её корнем.
