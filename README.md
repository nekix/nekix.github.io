# Работа с часовыми поясами в .NET: выбор между IANA и Windows Time Zones

> **Введение**: В этом материале рассматривается практический опыт реализации работы с часовыми поясами в .NET приложении с требованиями актуальности данных о часовых поясах, кроссплатформенности и глобализации.

Используемые по тексту понятия:

**ICU (International Components for Unicode)** — библиотека для локализации и глобализации, которая использует CLDR для данных о локализации и IANA для сведений о часовых поясах.

**IANA Time Zone Database** — стандартная база данных часовых поясов, поддерживаемая Internet Assigned Numbers Authority.

**CLDR (Common Locale Data Repository)** — репозиторий данных для локализации, включая информацию о часовых поясах.

**Windows Time Zones** — собственный формат часовых поясов Microsoft для операционной системы Windows.

Стояла задача реализовать в приложении работу с часовыми поясами и конвертацией времени. При этом нужно обеспечить:
- Работу с актуальными часовыми поясами (на момент работы приложения)
- Независимость работы от ОС
- Глобализацию

Microsoft предлагает использовать объекты `TimeZoneInfo`. Они инкапсулируют в себе информацию о часовом поясе и методы для преобразования времени одного часового пояса во время в другом часовом поясе. На первый взгляд звучит как то, что нам действительно нужно. Дело за малым: получить список часовых поясов, с которым мы будем работать, и определить, каким образом будет актуализироваться информация по самим часовым поясам. А вот тут начинаются нюансы, о которых я вас хочу предостеречь.

> **Примечание**: Azure предоставляет REST API для получения информации о timezone, в данном же материале речь идет именно о локальном работе:
> - [IANA Time Zones](https://learn.microsoft.com/en-us/rest/api/maps/timezone/get-timezone-enum-iana?view=rest-maps-2025-01-01&tabs=HTTP)
> - [Windows Time Zones](https://learn.microsoft.com/en-us/rest/api/maps/timezone/get-timezone-enum-windows?view=rest-maps-2025-01-01&tabs=HTTP)

## 1. Выбор стандарта часовых поясов

В Windows часовые пояса хранятся в своем формате и соответствуют именно набору часовых поясов, созданному Microsoft и поставляемому для Windows. В Linux и macOS используется Time Zone Database, поддерживаемая IANA. 

**Ключевая проблема**: Эти наборы **РАЗНЫЕ** (и не всегда взаимозаменяемые), но `TimeZoneInfo` позволяет работать с обоими. Наборы можно сопоставить — получается сопоставление, где одному Windows часовому поясу соответствует один или несколько часовых поясов IANA. Сопоставление поддерживается классом `TimeZoneInfo` и библиотекой ICU.

Это порождает вопрос: какой из наборов использовать для оперирования часовыми поясами, в том числе для показа пользователю?

IANA звучит для меня более предпочтительным выбором, так как:
- Является де-факто стандартом в отрасли
- Используется в большем стеке технологий  
- Дает большее число часовых поясов (привязок к регионам/городам)
- Поддерживается относительно Microsoft независимо

Стоит также отметить, что .NET позволяет использовать ICU, установленный в системе, либо ICU, поставляемый вместе с приложением в виде библиотеки `Microsoft.ICU.ICU4C.Runtime`. Это позволяет повысить автономность приложения, так как нам уже не нужно следить за обновлениями информации о часовых поясах в системе.

**Полезные ссылки**:
- [Date, Time, and Time Zone Enhancements in .NET 6](https://devblogs.microsoft.com/dotnet/date-time-and-time-zone-enhancements-in-net-6/#timezoneinfo-adjustmentrule-improvements)
- [App-local ICU](https://learn.microsoft.com/ru-ru/dotnet/core/extensions/globalization-icu#app-local-icu)

## 2. Получить список часовых поясов

Информация о часовых поясах подтягивается из ОС и её можно получить с помощью, например, `TimeZoneInfo.GetSystemTimeZones()`. Но в зависимости от ОС подтягивается **РАЗНАЯ** информация:

- **Windows**: часовые пояса загружаются из реестра `HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\Time Zones`
- **Linux и macOS**: из библиотеки ICU

Соответственно, мы получаем либо список Windows time zones, либо список IANA time zones в зависимости от ОС. И нет способа встроенными средствами в Windows получить список IANA time zones.

**Полезные ссылки**:
- [Windows/Linux/maxOS поведение] (https://learn.microsoft.com/en-us/dotnet/api/system.timezoneinfo.findsystemtimezonebyid?view=net-9.0#remarks)

- [TimeZoneInfo.GetSystemTimeZones()] (https://learn.microsoft.com/en-us/dotnet/api/system.timezoneinfo.getsystemtimezones?view=net-9.0)

### Варианты решения

#### 2.0. Ограничения стандартных средств

Получить гарантированно список time zone средствами `TimeZoneInfo` для IANA мы не можем. И стандартных средств платформы .NET (помимо низкоуровневых вызовов) нет. Microsoft хоть и используют ICU, но посчитали не нужным обернуть для .NET многие другие её полезные функции.

```csharp
// Пример низкоуровневого API из dotnet/runtime
[LibraryImport(Libraries.GlobalizationNative, EntryPoint = "GlobalizationNative_WindowsIdToIanaId", StringMarshalling = StringMarshalling.Utf16)]
internal static unsafe partial int WindowsIdToIanaId(string windowsId, IntPtr region, char* ianaId, int ianaIdLength);

[LibraryImport(Libraries.GlobalizationNative, EntryPoint = "GlobalizationNative_IanaIdToWindowsId", StringMarshalling = StringMarshalling.Utf16)]  
internal static unsafe partial int IanaIdToWindowsId(string ianaId, char* windowsId, int windowsIdLength);
```

С нативной реализацией низкоуровневого вызова в .NET можно ознакомиться в [репозитории](https://github.com/dotnet/runtime/blob/446ca5bb38e9fbc5706174a01f14bf3e400e16e1/src/native/libs/System.Globalization.Native/pal_timeZoneInfo.c#L41). Добавить функцию получения списка идентификаторов IANA было бы несложно.

#### 2.1. NodaTime

Достаточно много источников рекомендуют не париться и использовать эту библиотеку. Внутри поддерживает IANA tz db, но при этом вопросы локализации или, например, актуализации данных остаются открытыми. 

**Проблема**: Что делать, если ради работы с часовыми поясами нет желания тянуть альтернативный API для работы с датами и временем? В моем случае мне очень не понравилось, что только из-за невозможности получения IANA db списка time zone id придётся переходить в другую библиотеку.

#### 2.2. TimeZoneConverter

Автор библиотеки [TimeZoneConverter](https://github.com/mattjohnsonpint/TimeZoneConverter) предоставляет простой API для конвертации, получения списков идентификаторов часовых поясов. 

**Недостатки**:
- Библиотека эту информацию берет из CSV, распространяемых вместе с ней
- Для актуализации информации по базам данных часовых поясов необходимо обновить всю библиотеку
- Мы тянем за собой по факту дубликат данных, хранящихся в системе или в стандартном NuGet пакете `Microsoft.ICU.ICU4C.Runtime` или в системе
- TimeZoneConverter не отслеживает синхронизацию данных библиотеки и приложения/ОС, что может приводить к исключениям во время выполнения

#### 2.3. Нативный API и библиотека Icu.NET

В поисках альтернативы я наткнулся на библиотеку [icu-dotnet](https://github.com/sillsdev/icu-dotnet). Она представляет собой обертку над API для ICU (не над всем API). 

В частности, нас интересует `Icu.TimeZone.GetTimeZones(USystemTimeZoneType zoneType, string region)`. 

> **Важно**: Настоятельно рекомендую сравнивать функции обертки с документацией по ICU. Например, для `GetTimeZones` можно задать `region = null` для всех регионов — без просмотра документации и исходников ICU4C я бы этого сразу не понял.

```csharp
public IReadOnlyCollection<string> GetAvailableIanaIds()
{
    return Icu.TimeZone
        .GetTimeZones(USystemTimeZoneType.CanonicalLocation, null)
        .Select(tz => tz.Id)
        .OrderBy(id => id)
        .ToList();
}
```

**Проблемы**: В библиотеке также не реализована расширяемость, что сыграет злую шутку на этапе локализации.

**Преимущества**: Она использует именно встроенный в приложение или систему ICU, а не тянет за собой дубликат данных, что упрощает обновление и поддержку. Поэтому именно её я и выбрал.

## 3. Хранение списка часовых поясов

### 3.1. Сериализация TimeZoneInfo (не работает)

Изначально [идея была] (https://learn.microsoft.com/en-us/dotnet/standard/datetime/saving-and-restoring-time-zones) хранить список и сериализованные объекты часовых поясов в файлах ресурсов `.resx`. Для этого есть методы `TimeZoneInfo.ToSerializedString()` и `TimeZoneInfo.FromSerializedString(string)`.

> **Примечание**: Для тех, кто, как и я, искал альтернативу `ResXResourceReader` и `ResXResourceWriter`, могу посоветовать [KGySoft.CoreLibraries](https://github.com/koszeggy/KGySoft.CoreLibraries).

**Критическая проблема**: При попытке десериализации сериализованных строк часовых поясов IANA в Windows выбрасывается исключение. Эти проблемы уходят корнями в то, что ранее Microsoft разрабатывали `TimeZoneInfo` исключительно под Windows time zone db, что приводит к различию в формате хранения данных с IANA tz db.

Проблема не решается с 2017 года ([GitHub Issue #19794](https://github.com/dotnet/runtime/issues/19794#issuecomment-546429069)).

Представитель .NET команды советует сериализовать только идентификаторы часовых поясов ([комментарий](https://github.com/dotnet/runtime/issues/11421#issuecomment-454924834)).

В 2021 году тот же представитель сообщает, что **не рекомендуется использовать `ToSerializedString` в принципе**. Они планировали сделать этот API устаревшим. Все это говорит о ненадежности предложенного Microsoft способа хранения данных о временных зонах. Что поделать, бедная компания)

### 3.2. Хранение идентификаторов (рекомендуемый подход)

В таком случае можно хранить идентификаторы часовых поясов, а сами объекты `TimeZoneInfo` получать через `TimeZoneInfo.FindSystemTimeZoneById(string)`. 

Этот метод поддерживает создание как Windows, так и IANA time zones. Но при этом:
- В Linux, macOS нет информации о Windows time zones, и там это может не работать
- IANA есть как в Windows, так и в Linux, macOS

**Мое решение**: Я выбрал хранение в реляционной БД идентификаторов IANA. Это позволит прозрачно контролировать правильность используемых моим приложением идентификаторов часовых поясов в доменных моделях.

## 4. Локализация названия TimeZoneInfo (DisplayName)

`TimeZoneInfo` предоставляет свойство `DisplayName`, пригодное для отображения пользователю. Но предоставляет его в локали приложения или операционной системы.

Поскольку мне нужно было использовать другую локаль (да и любую в принципе, независимо от локали приложения и системы), я обнаружил следующие варианты:

### 4.1. Библиотека TimeZoneNames

[TimeZoneNames](https://github.com/mattjohnsonpint/TimeZoneNames/tree/main) позволяет получить локализованное название часового пояса или локализованные названия по регионам.

**Недостатки**:
- Библиотека загружает данные в систему по сети, что может быть неподходящим вариантом
- Мы получаем ещё один источник истины о часовых поясах и локализации, повышаем шанс рассинхронизации источников данных
- В ICU вся эта информация уже содержится

### 4.2. ICU.NET и низкоуровневые вызовы

Поскольку на предыдущих этапах я выбрал icu-dotnet, то решил использовать его инфраструктуру для низкоуровневого вызова необходимой функции ICU. Ею оказалась `ucal_getTimeZoneDisplayName`.

Поскольку icu-dotnet не спроектирована для простого внешнего расширения (если только не делать fork, менять исходники и все такое), то пришлось немного повозиться.

** TODO Основная идея реализации** (упрощенная версия):

```csharp
private string? GetTimeZoneDisplayNameFromICU(string timeZoneId, DisplayNameType displayType, string locale)
{
    // Создание календаря с указанным часовым поясом
    IntPtr cal = CreateCalendar(timeZoneId, locale);
    
    try
    {
        // Получение локализованного названия через ICU API
        return GetLocalizedDisplayName(cal, displayType, locale);
    }
    finally
    {
        CloseCalendar(cal);
    }
}
```

TODO Ссылки на IANA API и куда смотерть

Использовал ICU функции:
- `ucal_open` — создание календаря
- `ucal_getTimeZoneDisplayName` — получение локализованного названия

> **Примечание**: В ICU предоставляется 2 основных подхода: через календарь (подход выше) и через TimeZone объект.

Благодаря этому удалось получить список IANA time zone ids, а также при необходимости доступ к другим (не реализованным в icu-dotnet или .NET) функциям.

### 4.3. Альтернативный API

Готовя материал, я обнаружил, что Microsoft уже реализовывает нативно внутри себя `GetTimeZoneDisplayName`. Теоретически, можно было бы использовать `GlobalizationNative` реализацию либо через низкоуровневый вызов, либо через Reflection. Но это все нужно дополнительно проверять.

**Плюсы**: Меньше низкоуровневой возни с нашей стороны и не нужно тянуть сторонних библиотек.

**Ссылки**:
- [Interop.TimeZoneInfo.cs](https://github.com/dotnet/runtime/blob/446ca5bb38e9fbc5706174a01f14bf3e400e16e1/src/libraries/Common/src/Interop/Interop.TimeZoneInfo.cs#L12)
- [pal_timeZoneInfo.c](https://github.com/dotnet/runtime/blob/446ca5bb38e9fbc5706174a01f14bf3e400e16e1/src/native/libs/System.Globalization.Native/pal_timeZoneInfo.c#L279)

## 5. Улучшения и рекомендации

### 5.1. Тестирование

По-хорошему, учитывая возможную нестабильность .NET `TimeZoneInfo` API и особенностей работы, необходимо создать хотя бы минимальный набор тестов для проверки, что с новой версией ничего не сломалось (в том числе с новой версией ICU).

**Предлагаю следующие тесты**:
- Метод получения списка идентификаторов часовых поясов IANA (работа без исключений, возврат валидных значений)
- Метод получения объекта `TimeZoneInfo` из всех Id, полученных из списка идентификаторов часовых поясов IANA
- Преобразование из заданной даты и времени в UTC и/или выбранного часового пояса в перечисленные из метода получения списка идентификаторов часовых поясов IANA

### 5.2. Кеширование TimeZoneInfo объектов

В данный момент нативно в .NET кешируются именно системные `TimeZoneInfo`, а если мы работаем в Windows, то у нас кешируются Windows time zones. То есть мы не можем быть уверены, будет ли использоваться кеш в нашем (IANA) случае или нет. Тогда нужно будет реализовать свое кеширование, возможно, с учётом динамического обновления ICU.

### 5.3. Стратегия обновлений ICU

В зависимости от условий развертывания и требований к программной системе может быть необходимо реализовать различные способы обновления ICU, будь то автоматический или "ручной". В любом случае было бы неплохо иметь возможность выбора способа обновления данных и его замены/расширения (что `TimeZoneInfo` нам вообще не дает).

## TODO Заключение

Microsoft не доделали тяп ляп бла бла
Предлагаю следующие подходы бла бла
Грустно, соблюдаем осторожность

## TODO Полезные ссылки

- [Date, Time, and Time Zone Enhancements in .NET 6](https://devblogs.microsoft.com/dotnet/date-time-and-time-zone-enhancements-in-net-6/#time-zone-display-names-on-linux-and-macos)
- [TimeZoneInfo Documentation](https://learn.microsoft.com/ru-ru/dotnet/api/system.timezoneinfo.getsystemtimezones?view=net-8.0)
- [Saving and Restoring Time Zones](https://learn.microsoft.com/ru-ru/dotnet/standard/datetime/saving-and-restoring-time-zones)
- [ICU Documentation](https://unicode-org.github.io/icu-docs/apidoc/dev/icu4c/ucal_8h.html#a07f43e461ca2cfbba1e6d2d0afb6abbe)
