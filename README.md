# Создание снимков экрана при помощи FASM

## Содержание
1. [Коротко о главном](#Коротко-о-главном)
2. [Что имеем](#Что-имеем)
3. [Структуры](#Структуры)

## Коротко о главном
Эта статья основана на коде **Томаша Грыштара** (Tomasz Grysztar) [[Ссылка на исходники](https://flatassembler.net/examples/screenshot.zip)] и написана исключительно в *образовательных целях*. Она объясняет работу ассемблерного кода, который создает снимки экрана в формате JPEG.

Все права на ассемблерный код, представленный в статье, принадлежат Томашу Грыштару.

## Что имеем
1. **Описание используемых структур**
    - GdiplusStartupInput
    - ImageCodecInfo
2. **Используемые API-функции**
    - GdiplusStartup
    - GdipGetImageEncodersSize
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

## Структуры

Программа представлена двумя структурами: **GdiplusStartupInput** и **ImageCodecInfo**.

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

Объявление структуры в FASM.
```ASM
struct GdiplusStartupInput
  ; Указывает версию GDI+
  GdiplusVersion           dd ?
  ; Указатель на функцию обратного вызова
  DebugEventCallback       dd ?
  ; Логическое значение, указывающее, следует ли подавлять фоновый поток GDI
  SuppressBackgroundThread dd ?
  ; Логическое значение, указывающее, необходимо ли, чтобы GDI+ подавлял внешние кодеки изображений
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
