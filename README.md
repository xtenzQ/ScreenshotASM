# Создание снимков экрана при помощи FASM

## Содержание
1. [Введение](#Введение)
2. [Секция импорта](#Секция-импорта)
3. [Структуры](#Структуры)
4. [API-функции](#api-функции)

## Введение
Эта статья основана на коде **Томаша Грыштара** (Tomasz Grysztar) [[Ссылка на исходники](https://flatassembler.net/examples/screenshot.zip)] и написана исключительно в *образовательных целях*. Она объясняет работу ассемблерного кода, который создает снимки экрана в формате JPEG.

Описание структур и API-функции преимущественно является переводом описаний с [MSDN](https://msdn.microsoft.com/).

[Пост на **FlatAssembler.net**](https://board.flatassembler.net/topic.php?t=10427)

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
- [VirtualAlloc](#virtualalloc)
- [GetImageEncoders](#getimageencoders)
- [VirtualFree](#virtualfree)
- [GetDC](#getdc)
- [GetSystemMetrics](#getsystemmetrics)
- [CreateCompatibleBitmap](#createcompatiblebitmap)
- [CreateCompatibleDC](#createcompatibledc)
- [SelectObject](#selectobject)
- [BitBlt](#bitblt)
- [GdipCreateBitmapFromHBITMAP](#gdipcreatebitmapfromhbitmap)
- [GdipSaveImageToFile](#gdipsaveimagetofile)
- [GdipDisposeImage](#gdipdisposeimage)
- [DeleteObject](#deleteobject)
- [ReleaseDC](#releasedc)
- [GdiplusShutdown](#gdiplusshutdown)
- [ExitProcess](#exitprocess)

### GdiPlusStartup
[[Описание на MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/ms534077(v=vs.85).aspx)]

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

Использование в коде:
```ASM
  invoke  GdiplusStartup, token, input, NULL
  test    eax, eax
  jnz     exit
```

### GdipGetImageEncodersSize
[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/ms534081(v=vs.85).aspx)]

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

Использование в коде:
```ASM
  invoke  GdipGetImageEncodersSize, encoders_count, encoders_size
  test    eax, eax
  jnz     gdiplus_shutdown
```

### VirtualAlloc
[[Описание на MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/aa366887(v=vs.85).aspx)]

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

Использование в коде:
```ASM
  invoke  VirtualAlloc, 0, [encoders_size], MEM_COMMIT, PAGE_READWRITE
  test    eax, eax
  jz      gdiplus_shutdown
  mov     ebx, eax
```

### GetImageEncoders
[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/ms534080(v=vs.85).aspx)]

Функция GetImageEncoders получает массив объектов типа ImageCodecInfo, которые содержат информацию о доступных кодировщиках изображений.

Синтаксис:
```Cpp
Status GetImageEncoders(
  __in   UINT numEncoders,
  __in   UINT size,
  __out  ImageCodecInfo *encoders
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
numEncoders [in] | UINT | Целое число, определяющее количество доступных кодировщиков изображений
size [in] | UINT | Целое число, которое задает размер в байтах массив объектов типа ImageCodecInfo
encoders [out] | ImageCodecInfo* | Указатель на буфер, который получает массив объектов типа ImageCodecInfo

Использование в коде:
```Cpp
  invoke  GdipGetImageEncoders, [encoders_count], [encoders_size], ebx
  test    eax, eax
  jnz     gdiplus_shutdown
```

### VirtualFree

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/aa366892(v=vs.85).aspx)]

Функция VirtualFree освобождает выделенную память. 

Синтаксис:
```Cpp
BOOL VirtualFree
(  
  LPVOID lpAddress,
  SIZE_T dwSize,	
  DWORD dwFreeType	
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
lpAddress | LPVOID | Базовый адрес
dwSize | SIZE_T | Размер, если мы будем использовать тип освобождения, как MEM_RELEASE, то размер должен быть установлен в 0.
encoders [out] | DWORD | Определяет, какая операция произойдет с памятью

Типы операций с памятью:

Название | Описание
-------- | --------
lpAddress | LPVOID | Базовый адрес
MEM_RELEASE	| Высвобождение используемой памяти
MEM_DECOMMIT	| Высвобождение зарезервированной памяти

Использование в коде:
```ASM
invoke  VirtualFree, ebx, 0, MEM_RELEASE
```

### GetDC

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/dd144871(v=vs.85).aspx)]

Функция GetDC идентифицирует контекст устройства для CWnd клиентской области если успешно иначе, возвращаемое значение NULL.

Синтаксис:
```Cpp
HDC GetDC(
  _In_ HWND hWnd
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
hWnd | HWND | Дескриптор окна, чей контекст устройства должен быть извлечен

Использование в коде:
```ASM
invoke  GetDC, HWND_DESKTOP
test    eax, eax
jz      gdiplus_shutdown
mov     esi, eax
```

### GetSystemMetrics

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/ms724385(v=vs.85).aspx)]

Функция GetSystemMetrics идентифицирует заданный системный показатель или настройку конфигурации системы.

Синтаксис:

```Cpp
int WINAPI GetSystemMetrics(
  _In_ int nIndex
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
nIndex | int | Идентификатор настройки

Использование в коде:
```ASM
invoke  GetSystemMetrics, SM_CYSCREEN
mov     [screen_height], eax
invoke  GetSystemMetrics,SM_CXSCREEN
mov     [screen_width], eax
```

### CreateCompatibleBitmap

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/dd183488(v=vs.85).aspx)]

Функция CreateCompatibleBitmap создает точечный рисунок, совместимый с контекстом устройства.

Синтаксис:

```Cpp
HBITMAP CreateCompatibleBitmap(
  _In_ HDC hdc,
  _In_ int nWidth,
  _In_ int nHeight
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
hdc [in] | HDC | Дескриптор контекста устройства
nWidth [in] | int | Ширина рисунка в пикселях
nHeight [in] | int | Высота рисунка в пикселях

Использование в коде:
```ASM
invoke  CreateCompatibleBitmap, esi, [screen_width], [screen_height]
test    eax, eax
jz      release_desktop_dc
mov     ebx, eax
```

### CreateCompatibleDC

[[Описание на MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/dd183489(v=vs.85).aspx)]

Функция CreateCompatibleDC создает контекст устройства памяти, который является совместимым с устройством, определенным pDC.

Синтаксис:
```Cpp
HDC CreateCompatibleDC(
  _In_ HDC hdc
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
hdc [in] | HDC | Указатель на контекст устройства

Использование в коде:
```ASM
invoke  CreateCompatibleDC, esi
test    eax, eax
jz      delete_bitmap
mov     edi, eax
```

### SelectObject

[[Описание на MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/dd162957(v=vs.85).aspx)]

Функция SelectObject выбирает объект в указанном контексте устройства. Новый объект заменяет предыдущий объект того же типа.

Синтаксис:
```Cpp
HGDIOBJ SelectObject(
  _In_ HDC     hdc,
  _In_ HGDIOBJ hgdiobj
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
hdc [in] | HDC | Указатель на контекст устройства
hgdiobj [in] |	HGDIOBJ |	Указатель на объект, который должен быть выбран

Использование в коде:
```ASM
invoke  SelectObject, edi, ebx
test    eax, eax
jz      delete_dc
```

### BitBlt

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/dd183370(v=vs.85).aspx)]

Функция BitBlt копирует точечный рисунок от исходного контекста устройства до этого текущего контекста устройства.

Синтаксис:

```Cpp
BOOL BitBlt(
  _In_ HDC   hdcDest,
  _In_ int   nXDest,
  _In_ int   nYDest,
  _In_ int   nWidth,
  _In_ int   nHeight,
  _In_ HDC   hdcSrc,
  _In_ int   nXSrc,
  _In_ int   nYSrc,
  _In_ DWORD dwRop
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
hdcDest [in]	| HDC | Указатель на контекст конечного устройства
nXDest [in]	| int | Определяет логическую x-координату левого верхнего угла прямоугольника адресата.
nYDest [in]	| int | Определяет логическую y-координату левого верхнего угла прямоугольника адресата.
nWidth [in]	| int | Определяет ширину (в логических модулях) прямоугольника адресата и исходного точечного рисунка.
nHeight [in]	| int | Определяет высоту (в логических модулях) прямоугольника адресата и исходного точечного рисунка.
hdcSrc [in]	| HDC | Указатель на объект HDC, который идентифицирует контекст устройства из которого точечный рисунок будет скопирован
nXSrc [in]	| int | Определяет логическую x-координату левого верхнего угла исходного точечного рисунка
nYSrc [in]	| int | Определяет логическую y-координату левого верхнего угла исходного точечного рисунка
dwRop [in]	| DWORD | Определяет растровую операцию, которую нужно выполнить

Использование в коде:
```ASM
invoke  BitBlt, edi, 0, 0, [screen_width], [screen_height], esi, 0, 0, SRCCOPY
test    eax, eax
jz      delete_dc
```

### GdipCreateBitmapFromHBITMAP

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/ms533971(v=vs.85).aspx)]

Функция GdipCreateBitmapFromHBITMAP создает точечное изображение на основе дескриптора GDI и дескриптора палитры GDI.

Синтаксис:

```Cpp
GpStatus WINGDIPAPI GdipCreateBitmapFromHBITMAP(
    HBITMAP hbm,
    HPALETTE hpal,
    GpBitmap** bitmap
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
hbm [in] | HBITMAP | Дескриптор точечного изображения
hgdiobj [in] | HPALETTE | Дескриптор палитры GDI
bitmap [out] | GpBitmap | Указатель на переменную DWORD, которая получает указатель на объект bitmap

Использование в коде:
```ASM
invoke  GdipCreateBitmapFromHBITMAP, ebx, NULL, gdip_bitmap
test    eax, eax
jnz     delete_dc
```

### GdipSaveImageToFile

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/ms534041(v=vs.85).aspx)]

Функция GdipSaveImageToFile сохраняет изображение в файл.

Синтаксис:

```Cpp
GpStatus WINGDIPAPI GdipSaveImageToFile(
    GpImage *image,
    GDIPCONST WCHAR* filename,
    GDIPCONST CLSID* clsidEncoder,
    GDIPCONST EncoderParameters* encoderParams
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
image [in] | GpImage | Указатель на изображение
filename [in] | GDIPCONST | Указатель на строку unicode, которая указывает имя пути для сохраненного изображения
clsidEncoder [in] | GDIPCONST | Указатель на CLSID, который указывает кодировщик, который будет использоваться для сохранения изображения
encoderParams [in] | GDIPCONST | Указатель на структуру EncoderParameters, которая содержит параметры, используемые кодировщиком

Использование в коде:
```ASM
invoke  GdipSaveImageToFile, [gdip_bitmap], file_name, encoder_clsid, NULL
```

### GdipDisposeImage

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/ms534041(v=vs.85).aspx)]

Функция GdipDisposeImage удаляет указанное изображение.

Синтаксис:
```Cpp
GpStatus WINGDIPAPI GdipDisposeImage(
    GpImage *image
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
image [in] | GpImage | Указатель на изображение

Использование в коде:
```ASM
invoke  GdipDisposeImage, [gdip_bitmap]
```

### DeleteObject

[[Описание на MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/dd183539(v=vs.85).aspx)]

Функция DeleteObject удаляет перо, кисть, шрифт, растровое изображение, область или палитру, освобождая все системные ресурсы, связанные с объектом. После удаления объекта указанный дескриптор более недействителен.

Синтаксис:
```Cpp
BOOL DeleteObject(
  _In_ HGDIOBJ hObject
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
hObject [in] | HGDIOBJ | Дескриптор объекта

Использование в коде:
```ASM
invoke  DeleteObject, edi
```

### ReleaseDC

[[Описание на MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/dd162920(v=vs.85).aspx)]

Функция ReleaseDC освобождает контекст устройства (DC) для его использования другими приложениями.

Синтаксис:
```Cpp
int ReleaseDC(
  _In_ HWND hWnd,
  _In_ HDC  hDC
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
hWnd [in] | HWND | Дескриптор окна, чей контекст должен быть высвобожден
hDC [in] | HDC | Дескриптор контекста устройства, который надо высвободить

Использование в коде:
```ASM
invoke  ReleaseDC, HWND_DESKTOP, esi
```

### GdiplusShutdown

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/library/windows/desktop/ms534076(v=vs.85).aspx)]

Функция GdiplusShutdown очищает ресурсы, используемые Windows GDI+. 

Синтаксис:
```Cpp
void GdiplusShutdown(
  __in  ULONG_PTR token
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
token [in] | ULONG_PTR | Токен, возвращенный предыдущим вызовом GdiplusStartup

Использование в коде:
```ASM
invoke  GdiplusShutdown, [token]
```

### ExitProcess

[[Описание на MSDN](https://msdn.microsoft.com/ru-ru/ms682658)]

Функция ExitProcess заканчивает работу процесса и всех его потоков. 

Синтаксис:
```Cpp
VOID ExitProcess(
  UINT uExitCode	
);
```

Аргументы функции:

Название | Тип | Описание
-------- | --- | --------
uExitCode [in] | UINT | Определяет код выхода для процесса, и для всех потоков, которые завершают работу в результате вызова этой функци

Использование в коде:
```ASM
invoke  ExitProcess, 0
```
