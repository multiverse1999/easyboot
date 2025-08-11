Wtyczki Easyboot
================

Domyślnie [Easyboot](https://gitlab.com/bztsrc/easyboot) uruchamia zgodne z Multiboot2 kernele w formatach ELF i PE z partycji
rozruchowej. Jeśli jądro używa innego formatu pliku, innego protokołu rozruchowego lub nie znajduje się na partycji rozruchowej,
będziesz potrzebować wtyczek na partycji rozruchowej. Znajdziesz je w katalogu [src/plugins](../../src/plugins).

[[_TOC_]]

Instalacja
----------

Aby zainstalować wtyczki, wystarczy je skopiować do katalogu określonego w parametrze `(indir)`, w podkatalogu `easyboot` obok
pliku menu.cfg.

Na przykład:
```
bootpart
|-- easyboot
|   |-- linux_x86.plg
|   |-- minix_x86.plg
|   `-- menu.cfg
|-- EFI
|   `-- BOOT
|-- kernel
`-- initrd
$ easyboot bootpart dysk.img
```

Kompilacja
----------

*Od początku było oczywiste, że ELF nie nadaje się do tego zadania. Jest zbyt rozdęty i zbyt skomplikowany. Początkowo chciałem użyć
struct exec (klasyczny format UNIX a.out), ale niestety współczesne łańcuchy narzędzi nie potrafią już tego tworzyć. Postanowiłem
więc stworzyć własny format i własny linker dla wtyczek.*

Możesz skompilować kod źródłowy wtyczki za pomocą dowolnego standardowego kompilatora krzyżowego ANSI C do pliku obiektu ELF, ale
wtedy będziesz musiał użyć linkera [plgld](../../src/misc/plgld.c), aby utworzyć końcowy plik binarny. Jest to niezależny od
architektury linker krzyżowy, który będzie działał niezależnie od kodu maszynowego, dla którego wtyczka została skompilowana.
Ostateczny plik .plg ma tylko ułamek rozmiaru pliku .o ELF, z którego został wygenerowany.

### Wtyczka API

Źródło C wtyczki musi zawierać plik nagłówkowy `src/loader.h` i musi zawierać wiersz `EASY_PLUGIN`. Ma on jeden parametr, typ
wtyczki, po którym następuje specyfikacja dopasowania identyfikatora. Ta ostatnia jest używana przez ładowarkę do określenia, kiedy
użyć danej wtyczki.

Na przykład:

```c
#include "../loader.h"

/* magiczne bajty identyfikujące jądro Linuxa */
EASY_PLUGIN(PLG_T_KERNEL) {
   /* przes.  rozm. typ            bajty magiczne */
    { 0x1fe,     2, PLG_M_CONST, { 0xAA, 0x55, 0, 0 } },
    { 0x202,     4, PLG_M_CONST, { 'H', 'd', 'r', 'S' } }
};

/* punkt wejścia, prototyp jest definiowany przez typ wtyczki */
PLG_API void _start(uint8_t *buf, uint64_t size);
{
    /* przygotuj środowisko dla jądra Linux */
}
```

Wtyczki mogą używać wielu zmiennych i funkcji, które są zdefiniowane w pliku nagłówkowym i powiązane ze środowiskiem wykonawczym.

```c
uint32_t verbose;
```
Poziom szczegółowości. Wtyczka może drukować dane wyjściowe tylko wtedy, gdy jest różna od zera, z wyjątkiem komunikatów o błędach.
Im większa jest jego wartość, tym więcej szczegółów powinno zostać wydrukowanych.

```c
uint64_t file_size;
```
Całkowity rozmiar otwartego pliku (patrz `open` i `loadfile` poniżej).

```c
uint8_t *root_buf;
```
Gdy wtyczka systemu plików jest inicjowana, zawiera ona pierwsze 128 kb partycji (mam nadzieję, że wliczając superblok). Później
wtyczka systemu plików może ponownie wykorzystać ten bufor 128 kb do dowolnego celu (pamięć podręczna FAT, pamięć podręczna inodów
itd.)

```c
uint8_t *tags_buf;
```
Zawiera znaczniki Multiboot2. Wtyczka jądra może je przeanalizować, aby przekonwertować dane dostarczone przez menedżera rozruchu
na dowolny format oczekiwany przez jądro. Ten wskaźnik wskazuje na początek bufora.

```c
uint8_t *tags_ptr;
```
Ten wskaźnik wskazuje koniec bufora tagów Multiboot2. Wtyczki tagów mogą dodawać tutaj nowe tagi i dostosowywać ten wskaźnik.

```c
uint8_t *rsdp_ptr;
```
Wskazuje na wskaźnik ACPI RSDP.

```c
uint8_t *dsdt_ptr;
```
Wskazuje na opis sprzętu DSDT (lub GUDT, FDT).

```c
efi_system_table_t *ST;
```
Na komputerach z UEFI wskazuje na tabelę systemową EFI, w przeciwnym wypadku `NULL`.

```c
void memset(void *dst, uint8_t c, uint32_t n);
void memcpy(void *dst, const void *src, uint32_t n);
int  memcmp(const void *s1, const void *s2, uint32_t n);
```
Obowiązkowe funkcje pamięci (kompilator C może generować wywołania tych funkcji, nawet jeśli nie ma bezpośredniego wywołania).

```c
void *alloc(uint32_t num);
```
Przydziela `num` stron (4k) pamięci. Wtyczki nie mogą przydzielać dużo, muszą dążyć do minimalnego wykorzystania pamięci.

```c
void free(void *buf, uint32_t num);
```
Zwolnij wcześniej przydzieloną pamięć `num` stron.

```c
void printf(char *fmt, ...);
```
Wyświetla sformatowany ciąg znaków w konsoli rozruchowej.

```c
uint64_t pb_init(uint64_t size);
```
Uruchamia pasek postępu, `size` to całkowity rozmiar, który reprezentuje. Zwraca liczbę bajtów wartych jeden piksel.

```c
void pb_draw(uint64_t curr);
```
Rysuje pasek postępu dla bieżącej wartości. `curr` musi zawierać się pomiędzy 0 a całkowitym rozmiarem.

```c
void pb_fini(void);
```
Zamyka pasek postępu, czyszcząc jego miejsce na ekranie.

```c
void loadsec(uint64_t sec, void *dst);
```
Używane przez wtyczki systemu plików, ładują sektor z dysku do pamięci. `sec` to numer sektora względem partycji głównej.

```c
void sethooks(void *o, void *r, void *c);
```
Używane przez wtyczki systemu plików, ustawia haki funkcji open / read / close dla systemu plików partycji głównej.

```c
int open(char *fn);
```
Otwórz plik na partycji głównej (lub rozruchowej) do odczytu, zwraca 1 w przypadku powodzenia. W danym momencie można otworzyć
tylko jeden plik. Jeśli wcześniej nie było wywołania `sethooks`, to działa na partycji rozruchowej.

```c
uint64_t read(uint64_t offs, uint64_t size, void *buf);
```
Odczytuje dane z otwartego pliku na pozycji wyszukiwania `offs` do pamięci i zwraca liczbę faktycznie odczytanych bajtów.

```c
void close(void);
```
Zamyka otwarty plik.

```c
uint8_t *loadfile(char *path);
```
Załaduj plik w całości z partycji root (lub boot) do nowo przydzielonego bufora pamięci i transparentnie go rozpakuj, jeśli
znaleziono wtyczkę. Rozmiar zwrócony w `file_size`.

```c
int loadseg(uint32_t offs, uint32_t filesz, uint64_t vaddr, uint32_t memsz);
```
Załaduj segment z bufora jądra. Sprawdza to, czy pamięć `vaddr` jest dostępna i mapuje segment, jeśli jest to wyższa połowa. `offs`
to przesunięcie pliku, więc względne do bufora jądra. Jeśli `memsz` jest większe niż `filesz`, różnica jest wypełniana zerami.

```c
void _start(void);
```
Punkt wejścia dla wtyczek systemu plików (`PLG_T_FS`). Powinien on analizować superblok w `root_buf` i wywołać `sethooks`. W
przypadku błędu powinien po prostu zwrócić bez ustawiania swoich haków.

```c
void _start(uint8_t *buf, uint64_t size);
```
Punkt wejścia dla wtyczek jądra (`PLG_T_KERNEL`). Otrzymuje obraz jądra w pamięci, powinien przenieść jego segmenty, skonfigurować
odpowiednie środowisko i przekazać kontrolę. Jeśli nie ma błędu, nigdy nie zwraca.

```c
uint8_t *_start(uint8_t *buf);
```
Punkt wejścia dla wtyczek dekompresora (`PLG_T_DECOMP`). Otrzymuje skompresowany bufor (i jego rozmiar w `file_size`) i powinien
zwrócić przydzielony nowy bufor z nieskompresowanymi danymi (i ustawić nowy rozmiar bufora również w `file_size`). Musi zwolnić
stary bufor (uwaga, `file_size` jest w bajtach, ale free() oczekuje rozmiaru w stronach). W przypadku błędu, `file_size` nie może
zostać zmieniony i musi zwrócić niezmodyfikowany oryginalny bufor.

```c
void _start(void);
```
Punkt wejścia dla wtyczek tagów (`PLG_T_TAG`). Mogą dodać nowe tagi w `tags_ptr` i dostosować ten wskaźnik do nowej, wyrównanej
do 8 bajtów pozycji.

### Funkcje lokalne

Wtyczki mogą używać funkcji lokalnych, jednak z powodu błędu CLang, *muszą* być zadeklarowane jako `static`. (Błąd polega na tym,
że CLang generuje rekordy PLT dla tych funkcji, mimo że flaga „-fno-plt” jest przekazywana w wierszu poleceń. Użycie `static`
obejść to).

Specyfikacja formatu pliku niskiego poziomu
-------------------------------------------

W przypadku, gdyby ktoś chciał napisać wtyczkę w języku innym niż C (na przykład w Asemblerze), poniżej znajduje się opis niskiego
poziomu formatu pliku.

Jest bardzo podobny do formatu a.out. Plik składa się z nagłówka o stałym rozmiarze, po którym następują sekcje o różnej długości.
Nie ma nagłówka sekcji, dane każdej sekcji bezpośrednio następują po danych poprzedniej sekcji w następującej kolejności:

```
(chodnikowiec)
(identyfikator rekordów zgodności)
(rekordy relokacji)
(kod maszynowy)
(dane tylko do odczytu)
(zainicjowane dane do odczytu i zapisu)
```

W przypadku pierwszej prawdziwej sekcji, kodu maszynowego, wyrównanie jest uwzględnione. W przypadku wszystkich pozostałych sekcji,
do rozmiaru poprzedniej sekcji dodawane jest wypełnienie.

WSKAZÓWKA: jeśli przekażesz wtyczkę jako pojedynczy argument do `plgld`, zrzuci ona sekcje pliku, wyświetlając dane wyjściowe
podobne do `readelf -a` lub `objdump -xd`.

### Chodnikowiec

Wszystkie liczby są podane w formacie little-endian, niezależnie od architektury.

| Offset  | Rozm. | Opis                                                           |
|--------:|------:|----------------------------------------------------------------|
|       0 |     4 | magiczne bajty `EPLG`                                          |
|       4 |     4 | całkowity rozmiar pliku                                        |
|       8 |     4 | całkowita ilość pamięci wymagana podczas ładowania pliku       |
|      12 |     4 | rozmiar sekcji kodu                                            |
|      16 |     4 | rozmiar sekcji danych tylko do odczytu                         |
|      20 |     4 | punkt wejścia wtyczki                                          |
|      24 |     2 | kod architektury (taki sam jak ELF)                            |
|      26 |     2 | liczba rekordów relokacji                                      |
|      28 |     1 | liczba rekordów pasujących do identyfikatora                   |
|      29 |     1 | najwyższy wpis GOT z referencjami                              |
|      30 |     1 | wersja formatu pliku (na razie 0)                              |
|      31 |     1 | typ wtyczki (1=system plików, 2=jądro, 3=dekompresor, 4=tag)   |

Kod architektury jest taki sam, jak w nagłówkach ELF, na przykład 62 = x86_64, 183 = Aarch64 i 243 = RISC-V.

Typ wtyczki określa prototyp punktu wejścia, ABI jest zawsze SysV.

### Sekcja dopasowania identyfikatora

Sekcja ta zawiera tyle z poniższych rekordów, ile określono w polu nagłówka „liczba rekordów pasujących do identyfikatora”.

| Offset  | Rozm. | Opis                                                     |
|--------:|------:|----------------------------------------------------------|
|       0 |     2 | offset                                                   |
|       2 |     1 | rozmiar                                                  |
|       3 |     1 | typ                                                      |
|       4 |     4 | magiczne bajty do dopasowania                            |

Najpierw początek tematu jest ładowany do bufora. Akumulator jest ustawiany, początkowo z 0. Przesunięcia w tych rekordach są zawsze
względne do tego akumulatora i odnoszą się do tego bajtu w buforze.

Pole typu mówi, jak interpretować przesunięcie. Jeśli jest to 1, to przesunięcie plus akumulator jest używane jako wartość. Jeśli
jest to 2, to 8-bitowa wartość bajtu jest pobierana w przesunięciu, 3 oznacza wzięcie 16-bitowej wartości słowa, a 4 oznacza
wzięcie 32-bitowej wartości DWORD. 5 oznacza wzięcie 8-bitowej wartości bajtu i dodanie do niej akumulatora, 6 oznacza wzięcie
16-bitowej wartości słowa i dodanie do niej akumulatora, a 7 jest takie samo, ale z wartością 32-bitową. 8 będzie wyszukiwać
magiczne bajty od bajtu akumulatora do końca bufora w krokach przesunięcia i jeśli zostanie znaleziony, zwróci pasujący przesunięcie
jako wartość.

Jeśli rozmiar jest zerowy, akumulator jest ustawiany na wartość. Jeśli rozmiar nie jest zerowy, sprawdzana jest taka liczba bajtów,
jeśli pasują do podanych bajtów magicznych.

Na przykład, aby sprawdzić, czy plik wykonywalny PE rozpoczyna się instrukcją NOP:
```c
EASY_PLUGIN(PLG_T_KERNEL) {
   /* przes.  rozm. typ            bajty magiczne */
    { 0,         2, PLG_M_CONST, { 'M', 'Z', 0, 0 } },      /* sprawdź magiczne bajty */
    { 60,        0, PLG_M_DWORD, { 0, 0, 0, 0 } },          /* pobierz przesunięcie nagłówka PE do akumulatora */
    { 0,         4, PLG_M_CONST, { 'P', 'E', 0, 0 } },      /* sprawdź magiczne bajty */
    { 40,        1, PLG_M_DWORD, { 0x90, 0, 0, 0 } }        /* sprawdź instrukcję NOP w punkcie wejścia */
};
```

### Sekcja relokacji

Sekcja ta zawiera tyle rekordów, ile określono w polu nagłówka „liczba rekordów relokacji”.

| Offset  | Rozm. | Opis                                                     |
|--------:|------:|----------------------------------------------------------|
|       0 |     4 | offset                                                   |
|       4 |     4 | rodzaj relokacji                                         |

Znaczenie bitów w typie:

| Od      | Do    | Opis                                                     |
|--------:|------:|----------------------------------------------------------|
|       0 |     7 | symbol (0 - 255)                                         |
|       8 |     8 | adresowanie względne PC                                  |
|       9 |     9 | adresowanie względne GOT pośrednie                       |
|      10 |    13 | natychmiastowy indeks maski (0 - 15)                     |
|      14 |    19 | bit startowy (0 - 63)                                    |
|      20 |    25 | bit koniec (0 - 63)                                      |
|      26 |    31 | zanegowana pozycja bitu flagi adresu (0 - 63)            |

Pole przesunięcia jest względne w stosunku do wartości magicznej w nagłówku wtyczki i wybiera liczbę całkowitą w pamięci, w której
ma zostać wykonana relokacja.

Symbol wskazuje, którego adresu użyć. 0 oznacza adres BASE, gdzie wtyczka została załadowana do pamięci, czyli adres magii nagłówka
w pamięci. Inne wartości wybierają zewnętrzny adres symbolu z GOT, zdefiniowany w programie ładującym lub w innej wtyczce, spójrz
na tablicę `plg_got` w źródle plgld.c, aby zobaczyć, która wartość odpowiada któremu symbolowi. Jeśli względny bit GOT wynosi 1,
używany jest adres wpisu GOT symbolu, zamiast rzeczywistego adresu symbolu.

Jeśli bit względny PC wynosi 1, to najpierw od adresu odejmuje się przesunięcie (tryb adresowania względnego wskaźnika instrukcji).

Indeks maski bezpośredniej mówi, które bity przechowują adres w instrukcji. Jeśli jest to 0, adres jest zapisywany „tak jak jest” do
przesunięcia, niezależnie od architektury. Dla x86_64 dozwolony jest tylko indeks 0. Dla ARM Aarch64: 0 = tak jak jest, 1 = 0x07ffffe0
(przesunięcie w lewo o 5 bitów), 2 = 0x07fffc00 (przesunięcie w lewo o 10 bitów), 3 = 0x60ffffe0 (w przypadku instrukcji ADR/ADRP
natychmiast jest przesuwany i dzielony na dwie grupy bitów). Przyszłe architektury mogą definiować więcej i różnych natychmiastowych
masek bitowych.

Używając maski natychmiastowej, bity end - start + 1 są pobierane z pamięci i rozszerzone ze znakiem. Ta wartość jest dodawana do
adresu (addend, a w przypadku wewnętrznych odniesień, wewnętrzny adres symbolu jest tutaj również kodowany).

Jeśli negowany bit flagi adresu nie jest 0 i adres jest dodatni, to ten bit jest czyszczony. Jeśli adres jest ujemny, to ten bit
jest ustawiany, a adres jest negowany.

Na koniec bity startu i końca wybierają, która część adresu ma zostać zapisana do wybranej liczby całkowitej. Definiuje to również
rozmiar relokacji, bity poza tym zakresem i te, które nie są częścią natychmiastowej maski, pozostają niezmienione.

### Sekcja kodu

Sekcja ta zawiera instrukcje maszynowe dla architektury określonej w nagłówku i ma liczbę bajtów równą liczbie podanej w polu
rozmiaru kodu.

### Sekcja danych tylko do odczytu

To jest opcjonalna sekcja, może jej brakować. Jest tak długa, jak wskazuje pole rozmiaru sekcji tylko do odczytu w nagłówku.
Wszystkie zmienne stałe są umieszczane w tej sekcji.

### Sekcja zainicjowanych danych

To jest opcjonalna sekcja, może jej brakować. Jeśli w pliku są jeszcze bajty po sekcji kodu (lub opcjonalnej sekcji danych tylko do
odczytu), to wszystkie te bajty są uważane za sekcję danych. Jeśli zmienna jest inicjowana wartością różną od zera, to jest
umieszczana w tej sekcji.

### Sekcja BSS

To jest opcjonalna sekcja, może jej brakować. Ta sekcja nigdy nie jest przechowywana w pliku. Jeśli pole rozmiaru w pamięci jest
większe niż pole rozmiaru pliku w nagłówku, ich różnica zostanie wypełniona zerami w pamięci. Jeśli zmienna nie jest zainicjowana
lub zainicjowana jako zero, jest ona umieszczana w tej sekcji.
