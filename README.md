# Создание снимков экрана при помощи FASM

## Содержание
1. [Коротко о главном](#Коротко-о-главном)
2. [Что имеем](#Что-имеем)
3. [Секция импорта](#Секция-импорта)
4. [Структуры](#Структуры)
5. [API-функции](#api-функции)

## Коротко о главном
Эта статья основана на коде **Томаша Грыштара** (Tomasz Grysztar) [[Ссылка на исходники](https://flatassembler.net/examples/screenshot.zip)] и написана исключительно в *образовательных целях*. Она объясняет работу ассемблерного кода, который создает снимки экрана в формате JPEG.

Все права на ассемблерный код, представленный в статье, принадлежат Томашу Грыштару.

Описание структур и API-функции преимущественно является переводом описаний с [MSDN](https://msdn.microsoft.com/).

## Что имеем

## Секция импорта

```ASM
section '.rdata' data readable

; импортируемые функции и библиотеки
data import
  library                           \
    kernel32,       'KERNEL32.DLL', \
    user32,         'USER32.DLL',   \
    gdi32,          'GDI32.DLL',    \
    gdiplus,        'GDIPLUS.DLL'

  import  gdiplus,                                               \
    GdiplusStartup,               'GdiplusStartup',              \
    GdiplusShutdown,              'GdiplusShutdown',             \
    GdipGetImageEncodersSize,     'GdipGetImageEncodersSize',    \
    GdipGetImageEncoders,         'GdipGetImageEncoders',        \
    GdipSaveImageToFile,          'GdipSaveImageToFile',         \
    GdipDisposeImage,             'GdipDisposeImage',            \
    GdipCreateBitmapFromHBITMAP,  'GdipCreateBitmapFromHBITMAP'

  include 'api/kernel32.inc'
  include 'api/user32.inc'
  include 'api/gdi32.inc'
end data
```

## Структуры

Программа представлена двумя структурами:
- [GdiplusStartupInput](#gdiplusstartupinput)
- [ImageCodecInfo](#imagecodecinfo)

### GdiplusStartupInput
[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/ms534067(v=vs.85).aspx)]

Структура, содержащая блок аргументов, требуемых для функции GdiplusStartup.

Синтаксис:
```Cpp
typedef struct GdiplusStartupInput {
  UINT32         GdiplusVersion;
  DebugEventProc DebugEventCallback;
  BOOL           SuppressBackgroundThread;
  BOOL           SuppressExternalCodecs;
};
```

Объявление структуры в FASM:
```ASM
struct GdiplusStartupInput
  GdiplusVersion           dd ?
  DebugEventCallback       dd ?
  SuppressBackgroundThread dd ?
  SuppressExternalCodecs   dd ?
ends
```

Члены структуры представлены в таблице ниже:

Название | Тип | Описание
-------- | --- | --------
GdiplusVersion | UINT32 | Указывает версию GDI+. Должна быть 1.
DebugEventCallback | DebugEventProc | Указатель на функцию обратного вызова, которую GDI+ может вызывать. Значение по умолчанию - NULL.
SuppressBackgroundThread | BOOL | Логическое значение, указывающее, следует ли подавлять фоновый поток GDI+
SuppressExternalCodecs | BOOL | Логическое значение, указывающее, необходимо ли, чтобы GDI+ подавлял внешние кодеки изображений.

Структура **GdiplusStartupInput** предоставляет конструктор, который устаналивает член структуры **GdiplusVersion** в **1** и позволяет указать значения для остальных трех членов. Все параметры конструктора являются необязательными, поэтому можно объявить перменную типа **GdiplusStartupInput** без передачи каких-либо аргументов конструктору, и все члены будут инициализированы соответствующими значениями по умолчанию.

### ImageCodecInfo
[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/system.drawing.imaging.imagecodecinfo(v=vs.110).aspx)]

Структура ImageCodecInfo предоставляет информацию об установленных кодеках изображения.

Синтаксис:
```Cpp
typedef struct ImageCodecInfo {
   CLSID ClassID;
   GUID FormatID;
   WCHAR * CodecName;
   WCHAR * DllName;
   WCHAR * FormatDescription;
   WCHAR * FilenameExtension;
   WCHAR * MimeType;
   DWORD Flags;
   DWORD Version;
   DWORD SigCount;
   DWORD SigSize;
   BYTE * SigPattern;
   BYTE * SigMask;
};
```

Объявление структуры в FASM:
```ASM
struct ImageCodecInfo
  Clsid             db 16 dup ?
  FormatID          db 16 dup ?
  CodecName         dd ?
  DllName           dd ?
  FormatDescription dd ?
  FilenameExtension dd ?
  MimeType          dd ?
  Flags             dd ?
  Version           dd ?
  SigCount          dd ?
  SizeSize          dd ?
  SigPattern        dd ?
  SigMask           dd ?
ends
```
Члены структуры представлены в таблице ниже:

Название | Тип | Описание
-------- | --- | --------
Clsid | CLSID | Идентификатор кодека
FormatID | GUID | Идентификатор формата файла
CodecName | WCHAR | Указатель строку, содержащую имя кодека.
DllName | WCHAR | Указатель на строку, содержащую имя пути к DLL, в котором находится кодек
FormatDescription | WCHAR | Указатель на строку, которая содержит имя формата файла, используемого кодеком
FilenameExtension | WCHAR | Указатель на строку, которая содержит все расширения имен файлов, связанные с кодеком
MimeType | WCHAR | Указатель на строку, которая содержит тип mime-кодека.
Flags | DWORD | Комбинация флагов из перечисления ImageCodecFlags
Version | DWORD | Целое число, указывающее версию кодека
SigCount | DWORD | Целое число, которое указывает количество подписей, используемых файловым форматом, связанным с кодеком
SizeSize | DWORD | Целое число, указывающее количество байтов в каждой подписи
SigPattern | BYTE | Указатель на массив байтов, содержащий шаблон для каждой подписи
SigMask | BYTE | Указатель на массив байтов, содержащий маску для каждой подписи


## API-функции

API-функции, использованные в программе:
- [GdiplusStartup](#gdiplusstartup)
- [GdipGetImageEncodersSize](#gdipgetimageencoderssize)
- VirtualAlloc
- GdipGetImageEncoders
- VirtualFree
- GetDC 
- GetSystemMetrics
- CreateCompatibleBitmap
- CreateCompatibleDC
- SelectObject
- BitBlt
- GdipCreateBitmapFromHBITMAP
- GdipSaveImageToFile
- GdipDisposeImage
- DeleteObject
- ReleaseDC
- GdiplusShutdown
- ExitProcess

### GdiPlusStartup
Функция **GdiplusStartup** инициализирует Windows GDI+.
```Cpp
Status GdiplusStartup(
  __out  ULONG_PTR token *token,
  __in   const GdiplusStartupInput *input,
  __out  GdiplusStartupOutput *output
);
```

Аргументы функции GdiplusStartup:

Название | Тип | Описание
-------- | --- | --------
token [out] | ULONG_PTR token* | Указатель на ULONG_PTR, который получает токен
input [in] | const | GdiPlusStartupInput* | Указатель на структуру GdiplusStartupInput
output [out] | GdiplusStartupOutput* | Указатель на структуру GdiplusStartupOutput

### GdipGetImageEncodersSize
Функция GdipGetImageEncodersSize получает число доступных кодировщиков изображения и размер массива объектов типа ImageCodecInfo, который возвращает функция GetImageEncoders.

Синтаксис:
```Cpp
Status GetImageEncodersSize(
  __out  UINT *numEncoders,
  __out  UINT *size
);
```

Аргументы функции GdipGetImageEncodersSize:

Название | Тип | Описание
-------- | --- | --------
numEncoders [out]	| UINT*	| Указатель на UINT, который получает число доступных кодировщиков изображения
size [out] | UINT* | Указатель на UINT, который получает размер массива объектов ImageCodecInfo (в байтах), возвращаемых функцией GetImageEncoders

### VirtualAlloc
С помощью данной функции есть возможность выделить или зарезервировать страницы виртуальной памяти. Между выделением и резервированием есть разница. При выделении физически выделяется память и естественно увеличивается файл подкачки. При резервировании этого не происходит. 

Синтаксис:
```Cpp
LPVOID WINAPI VirtualAlloc(
  _In_opt_ LPVOID lpAddress,
  _In_     SIZE_T dwSize,
  _In_     DWORD  flAllocationType,
  _In_     DWORD  flProtect
);
```
Аргументы функции VirtualAlloc:

Название | Тип | Описание
-------- | --- | --------
lpAddress [in, optional] | LPVOID | Базовый адрес
size [out] | SIZE_T | Размер области в байтах
flAllocationType [in] | DWORD | Способ получения
flProtect [in] | DWORD | Тип доступа

Способ получения:

Название | Описание
-------- | --------
MEM_COMMIT | Выполняется выделение страниц памяти для непосредственной работы с ними. Выделенные страницы заполняются нулями.
MEM_RESERVE | Резервирует страницы памяти

Тип доступа:

Название | Описание
-------- | --------
PAGE_READONLY | Только чтение
PAGE_READWRITE | Чтение и запись
PAGE_EXECUTE | Только исполнение программного кода
PAGE_EXECUTE_READ | Исполнение и чтение
PAGE_EXECUTE_READWRITE | Исполнение, чтение и запись
PAGE_GUARD | Сигнализация доступа к странице
PAGE_NOACCESS | Запрещен любой вид доступа
PAGE_NOCACHE | Отмена кэширования для страницы памяти
