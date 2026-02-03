```java

Общая цепочка (всегда одинаковый старт и финал)
1) buildRange(start, numDays, isForward, daysClassificationRaw)

Парсит daysClassificationRaw → DaysClassification classification

Считает step = +1/-1 через step(isForward)

В зависимости от classification строит список дат:

ALL → buildAllDates(...)

WORK → buildWorkDates(...)

Когда итоговый список дат готов, делает финальную загрузку МД ровно под него:

mdByShort = mdLoader.loadIndexByDateShort(requestedRange)

Собирает ответ по спеке:

toSortedRangeResponse(requestedRange, mdByShort)

Ветка ALL (простая)
2) buildAllDates(start, numDays, step)

Вычисляет конечную дату:

end = start + step * (numDays - 1)

Возвращает все даты между start и end включительно:

inclusiveRange(start, end)

3) inclusiveRange(a, b)

Делает start = min(a,b), end = max(a,b) — чтобы всегда вернуть по возрастанию, даже если isForward=false

В цикле добавляет даты от start до end включительно (+1 день)

✅ На этом итоговый диапазон готов → возвращаемся в buildRange(...) и идём в финальную загрузку МД + сборку ответа.

Ветка WORK (оконная, потому что конец заранее неизвестен)
2) buildWorkDates(start, numDays, step)

Задача: найти такую конечную дату, чтобы в непрерывном промежутке от start набралось numDays рабочих.

Берём стартовый размер окна:

windowSize = workWindowPolicy.initialWindowSize(numDays)

В цикле:

строим окно дат:

windowDates = generateWindow(start, windowSize, step)

батчем грузим МД под окно:

windowMd = mdLoader.loadIndexByDateShort(windowDates)

пытаемся внутри окна найти дату, на которой набралось numDays рабочих:

endDateOpt = tryFindEndDateByWorkDays(windowDates, windowMd, numDays)

если нашли:

возвращаем итоговый непрерывный диапазон: inclusiveRange(start, endDate)

если не нашли — увеличиваем окно через политику:

windowSize = workWindowPolicy.nextWindowSize(windowSize)

Если окно доросло до max и всё равно не нашли — кидаем ошибку (защитный сценарий)

3) generateWindow(start, windowSize, step)

Строит список дат длиной windowSize в направлении step:

start

start+step

start+2*step

…

⚠️ Важно: порядок тут “в направлении шага”, потому что по нему мы считаем рабочие дни.

4) mdLoader.loadIndexByDateShort(windowDates)

Внутри:

toDistinctShortKeys(windowDates) → превращает даты в "dd.MM.yyyy" и убирает дубли

prodCalendDateService.searchByDates(keys) → один батч-запрос в МД (и кэш)

indexByDateShort(dtos) → делает Map<dateShort, CalendarDateDto>

То есть на выходе у тебя O(1) доступ:

“дай инфу по 03.02.2026” → map.get("03.02.2026")

5) tryFindEndDateByWorkDays(orderedWindowDates, mdByShort, targetWorkDays)

Проходит окно по порядку и считает рабочие:

workCount = 0

Для каждой даты d:

dto = requireDto(d, mdByShort) (проверка что МД вернуло запись)

если isWork(dto.getDateType()) → workCount++

если workCount == targetWorkDays → return Optional.of(d)

Если не набрали — Optional.empty()

6) requireDto(date, mdByShort)

Делает ключ "dd.MM.yyyy"

Берёт dto = map.get(key)

Если dto == null → ошибка (значит МД не вернуло дату, а мы без неё не можем корректно посчитать диапазон/сформировать ответ)

Финал для обеих веток: сборка ответа
7) toSortedRangeResponse(rangeDates, mdByShort)

Делает sorted = rangeDates.stream().sorted() — спека требует всегда по возрастанию

На каждую дату:

mdDto = requireDto(d, mdByShort)

собираем CalendarRangeItemDto:

id = mdDto.getId()

date = mdDto.getDateShort() (строго dd.MM.yyyy)

dateType = mdDto.getDateType()

dailyStatus = isWork(dateType) ? "рабочий день" : "не рабочий день"

Почему в WORK нельзя просто “один запрос”

Потому что конец диапазона неизвестен заранее: он зависит от того, сколько рабочих дней встретится, а это можно определить только после получения dateType на конкретные даты.
И раз в МД нет запроса по диапазону (только по списку slug), приходится:

либо дёргать МД по 1 дате (плохо),

либо грузить пачками окнами (хорошо, особенно с кэшем).
```
