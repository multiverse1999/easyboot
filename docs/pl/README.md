Uruchamianie jąder za pomocą Easyboot
=====================================

[Easyboot](https://gitlab.com/bztsrc/easyboot) to wielofunkcyjny menedżer rozruchu i program do tworzenia obrazów dysków startowych,
który potrafi ładować różne jądra systemów operacyjnych, w tym jądra zgodne ze standardem Multiboot2, w różnych formatach.

[[_TOC_]]

Instalacja
----------

```
 easyboot [-v|-vv] [-s <mb>] [-b <mb>] [-u <guid>] [-p <t> <u> <i>] [-e] [-c] <indir> <outfile|device>

  -v, -vv         zwiększyć gadatliwość/weryfikację
  -s <mb>         ustaw rozmiar obrazu dysku w megabajtach (domyślnie 35M)
  -b <mb>         ustaw rozmiar partycji rozruchowej w megabajtach (domyślnie 33M)
  -u <guid>       ustaw unikalny identyfikator partycji rozruchowej (domyślnie losowy)
  -p <t> <u> <i>  dodaj dodatkową partycję (typ guid, unikalny guid, plik obrazu)
  -e              dodaj El Torito Boot Catalog (obsługa rozruchu BIOS / EFI CDROM)
  -c              zawsze twórz nowy plik obrazu, nawet jeśli istnieje
  <indir>         użyj zawartości tego katalogu jako partycji rozruchowej
  <outfile>       obraz wyjściowy lub nazwa pliku urządzenia
```

Narzędzie **Easyboot** tworzy obraz dysku rozruchowego o nazwie `(outfile)` przy użyciu tabeli partycjonowania GUID z pojedynczą
partycją sformatowaną jako FAT32 i nazwaną jako „Partycja systemowa EFI” (w skrócie ESP). Zawartość tej partycji jest pobierana z
`(indir)`, którą podajesz. Musisz umieścić prosty plik konfiguracyjny w zwykłym tekście w tym katalogu o nazwie `easyboot/menu.cfg`.
Z końcówkami wierszy NL lub CRLF możesz łatwo edytować go w dowolnym edytorze tekstu. W zależności od konfiguracji możesz również
potrzebować [wtyczek](plugins.md) w tym katalogu o nazwie `easyboot/*.plg`.

Obraz można uruchomić także na Raspberry Pi i będzie działać w qemu. Jednak aby uruchomić system na prawdziwym komputerze, potrzebne
będą dodatkowe pliki oprogramowania układowego: `bootcode.bin`, `fixup.dat`, `start.elf` i plik .dtb w katalogu `(indir)`. Można je
pobrać z [oficjalnego repozytorium](https://github.com/raspberrypi/firmware/tree/master/boot) Raspberry Pi.

Narzędzie ma również kilka opcjonalnych flag wiersza poleceń: `-s (mb)` ustawia całkowity rozmiar wygenerowanego obrazu dysku w
megabajtach, podczas gdy `-b (mb)` ustawia rozmiar partycji rozruchowej w megabajtach. Oczywiście pierwsza musi być większa od
drugiej. Jeśli nie określono, rozmiar partycji jest obliczany na podstawie rozmiaru danego katalogu (minimum 33 Mb, najmniejszy
możliwy FAT32), a rozmiar dysku domyślnie jest większy o 2 Mb (z powodu wyrównania i miejsca potrzebnego na tabelę partycjonowania).
Jeśli między tymi dwiema wartościami rozmiaru jest różnica większa niż 2 Mb, możesz użyć narzędzi innych firm, takich jak `fdisk`,
aby dodać więcej partycji do obrazu według własnego uznania (lub zobacz `-p` poniżej). Jeśli chcesz uzyskać przewidywalny układ,
możesz również określić unikalny identyfikator partycji rozruchowej (UniquePartitionGUID) za pomocą flagi `-u <guid>`.

Opcjonalnie możesz dodać dodatkowe partycje za pomocą flagi `-p`. Wymaga to 3 argumentów: (PartitionTypeGUID), (UniquePartitionGUID)
i nazwy pliku obrazu zawierającego zawartość partycji. Tę flagę można powtarzać wielokrotnie.

Flaga `-e` dodaje katalog rozruchowy El Torito do wygenerowanego obrazu, dzięki czemu można go uruchomić nie tylko z pamięci USB,
ale także z napędu CDROM BIOS / EFI.

Jeśli `(outfile)` jest plikiem urządzenia (np. `/dev/sda` w systemie Linux, `/dev/disk0` w systemie BSD i `\\.\PhysicalDrive0` w
systemie Windows), to nie tworzy GPT ani ESP, zamiast tego lokalizuje już istniejące na urządzeniu. Nadal kopiuje wszystkie pliki
w `(indir)` do partycji rozruchowej i instaluje programy ładujące. Działa to również, jeśli `(outfile)` jest plikiem obrazu, który
już istnieje (w takim przypadku użyj flagi `-c`, aby zawsze najpierw utworzyć nowy, pusty plik obrazu).

Konfiguracja
------------

Plik `easyboot/menu.cfg` może zawierać następujące wiersze (bardzo podobne do składni grub.cfg, przykładowy plik konfiguracyjny
można znaleźć [tutaj](menu.cfg)):

### Uwagi

Wszystkie wiersze zaczynające się od znaku `#` są uważane za komentarze i pomijane do końca wiersza.

### Poziom szczegółowości

Poziom szczegółowości można ustawić w wierszu zaczynającym się od `verbose`.

```
verbose (0-3)
```

Informuje ładowarkę, ile informacji ma wydrukować na konsoli rozruchowej. Verbose 0 oznacza całkowitą ciszę (domyślnie), a verbose 3
zrzuci załadowane segmenty jądra i kod maszynowy w punkcie wejścia.

### Bufor ramki

Możesz poprosić o konkretną rozdzielczość ekranu za pomocą wiersza zaczynającego się od `framebuffer`. Format jest następujący:

```
framebuffer (szerokość) (wysokość) (bity na piksel) [#(kolor pierwszego planu)] [#(kolor tła)] [#(pasek postępu)]
```

**Easyboot** skonfiguruje bufor ramki, nawet jeśli ta linia nie istnieje (800 x 600 x 32bpp domyślnie). Ale jeśli ta linia istnieje,
spróbuje ustawić określoną rozdzielczość. Tryby paletowe nie są obsługiwane, więc liczba bitów na piksel musi wynosić co najmniej 15.

Jeśli podano czwarty opcjonalny parametr koloru, musi on być w notacji HTML, zaczynając od znaku hash, po którym następuje 6 cyfr
szesnastkowych, CCZZNN. Na przykład `#ff0000` to pełna jasna czerwień, a `#007f7f` to ciemniejszy cyjan. Ustawia on pierwszy plan
lub innymi słowy kolor czcionki. Podobnie, jeśli podano piąty opcjonalny parametr koloru, musi on być również w notacji HTML i
ustawia kolor tła. Ostatni opcjonalny parametr koloru jest podobny i ustawia kolor tła paska postępu.

Jeśli nie podano kolorów, domyślnie wyświetlane są białe litery na czarnym tle, a tło paska postępu jest ciemnoszare.

### Domyślna opcja rozruchu

Wiersz zaczynający się od `default` określa, który wpis menu powinien zostać uruchomiony bez interakcji użytkownika po upływie
określonego czasu.

```
default (indeks wpisu menu) (limit czasu msec)
```

Indeks menu to liczba od 1 do 9. Limit czasu jest podawany w milisekundach (jednej tysięcznej sekundy), więc 1000 oznacza jedną
sekundę.

### Wyrównanie menu

Wiersze zaczynające się od `menualign` zmieniają sposób wyświetlania opcji rozruchu.

```
menualign ("vertical"|"horizontal")
```

Domyślnie menu jest poziome (`horizontal`), ale można je zmienić na pionowe (`vertical`).

### Opcje rozruchu

Możesz określić do 9 pozycji menu z wierszami zaczynającymi się od `menuentry`. Format jest następujący:

```
menuentry (ikona) [etykieta]
```

Dla każdej ikony musi istnieć plik `easyboot/(ikona).tga` na partycji rozruchowej. Obraz musi być w formacie
[Targa format](../en/TGA.txt) zakodowanym długością runa, mapowanym kolorami, ponieważ jest to najbardziej skompresowana wersja
(pierwsze trzy bajty pliku muszą być `0`, `1` i `9` w tej kolejności, patrz typ danych 9 w specyfikacji). Jego wymiary muszą wynosić
dokładnie 64 piksele wysokości i 64 piksele szerokości.

Aby zapisać w tym formacie z GIMP-a, najpierw wybierz „Image > Mode > Indexed...”, w oknie pop-up ustaw „Maximum number of colors”
na 256. Następnie wybierz „File > Export As...”, wpisz nazwę pliku kończącą się na `.tga` i w oknie pop-up zaznacz „RLE compression”.
Jako narzędzie do konwersji wiersza poleceń możesz użyć ImageMagick, `convert (dowolny plik obrazu) -colors 256 -compress RLE ikona.tga`.

Opcjonalny parametr etykiety służy do wyświetlania informacji o wersji lub wydaniu ASCII w menu, a nie do dowolnych ciągów znaków,
dlatego w celu zaoszczędzenia miejsca UTF-8 nie jest obsługiwany, chyba że podasz również `easyboot/font.sfn`. (Czcionka UNICODE
wymaga dużej ilości pamięci, nawet jeśli [Scalable Screen Font](https://gitlab.com/bztsrc/scalable-font2) jest bardzo wydajna, to
i tak zajmuje około 820 KB. Dla porównania, plik unicode.pf2 GRUB-a jest znacznie większy, ma około 2392 KB, chociaż obie czcionki
zawierają około 55600 glifów w bitmapach 8x16 i 16x16. Wbudowana czcionka ASCII ma tylko bitmapy 8x16 i 96 glifów.)

Wszystkie wiersze, które następują po wierszu `menuentry`, będą należeć do tego menuentry, z wyjątkiem sytuacji, gdy ten wiersz jest
innym wierszem menuentry. Dla wygody możesz używać bloków, jak w GRUB, ale są one tylko składniowym lukrem. Nawiasy klamrowe są
traktowane jako znaki odstępu. Możesz je pominąć i zamiast tego używać tabulatorów, tak jak w skrypcie Pythona lub Makefile, jeśli
wolisz.

Na przykład
```
menuentry FreeBSD backup
{
    kernel bsd.old/boot
}
```

### Wybierz partycję

Linia zaczynająca się od `partition` wybiera partycję GPT. Musi być poprzedzona linią `menuentry`.

```
partition (unikalny UUID partycji)
```

Ta partycja będzie używana jako główny system plików dla opcji rozruchu. Zarówno jądro, jak i moduły zostaną załadowane z tej
partycji i w zależności od protokołu rozruchu, zostaną również przekazane do jądra. Określona partycja może znajdować się na
innym dysku niż dysk rozruchowy, **Easyboot** będzie iterował po wszystkich dyskach partycjonowanych GPT podczas rozruchu, aby
ją zlokalizować.

Dla wygody partycja jest również wyszukiwana w wierszu `kernel` (patrz poniżej). Jeśli dany wiersz polecenia rozruchu zawiera
ciąg `root=(UUID)` lub `root=*=(UUID)`, nie ma potrzeby tworzenia osobnego wiersza `partition`.

Jeśli partycja nie jest wyraźnie określona, ​​wówczas jądro i moduły są ładowane z partycji rozruchowej.

### Określ jądro

Linia zaczynająca się od `kernel` mówi, jaki plik powinien zostać uruchomiony i z jakimi parametrami. Musi być poprzedzona linią
`menuentry`.

```
kernel (ścieżka do pliku jądra) (opcjonalne argumenty wiersza poleceń rozruchu)
```

Ścieżka musi wskazywać na istniejący plik, wykonywalny plik binarny jądra i musi być absolutną ścieżką UTF-8 na partycji głównej
(lub rozruchowej). Jeśli jądro nie znajduje się w katalogu głównym partycji, separatorem katalogu jest zawsze `/`, nawet w
systemach UEFI. Jeśli nazwa zawiera spację, musi być ona poprzedzona znakiem `\`. Po ścieżce mogą następować argumenty wiersza
poleceń, oddzielone spacją. W przypadku jąder zgodnych z Multiboot2 te argumenty wiersza poleceń zostaną przekazane w znaczniku
*Wiersz poleceń rozruchowych* (typ 1). Nie zostaną one zmienione przez **Easyboot**, ani nie zostaną przeanalizowane, z wyjątkiem
przeszukania pod kątem specyfikatora partycji.

Domyślnie **Easyboot** może uruchamiać zgodne z Multiboot2 kernele w formatach ELF64 i PE32+/COFF (a w systemach UEFI również
aplikacje UEFI). Zwykle ten protokół nie zezwala na kernele wyższej połowy, ale **Easyboot** narusza protokół w sposób, który
nie psuje normalnych kerneli, które nie są kernelami wyższej połowy. Jeśli chcesz uruchomić jakikolwiek inny kernel, będziesz
potrzebować ładowarki kernela [wtyczka](plugins.md).

UWAGA: w przeciwieństwie do GRUB-a, gdzie trzeba używać specjalnych poleceń, takich jak „linux” lub „multiboot”, aby wybrać protokół
rozruchowy, tutaj wystarczy jedno polecenie, a protokół jest automatycznie wykrywany na podstawie jądra w czasie wykonywania.

### Ładowanie kolejnych modułów

Możesz załadować dowolne pliki (początkowe dyski RAM, sterowniki jądra itd.) wraz z jądrem, używając wierszy zaczynających się od
`module`. Musi być poprzedzony wierszem `menuentry`. Zauważ, że ten wiersz może być powtarzany wielokrotnie w każdym menuentry.
Jeśli protokół rozruchowy obsługuje initrd, to pierwszy wiersz `module` jest uważany za initrd.

```
module (ścieżka do pliku) (opcjonalne argumenty wiersza poleceń modułu)
```

Ścieżka musi wskazywać na istniejący plik i musi być absolutną ścieżką UTF-8 na partycji głównej (lub rozruchowej). Mogą po niej
następować argumenty wiersza poleceń, rozdzielone spacją. Jeśli plik jest skompresowany i istnieje dla niego dekompresja
[wtyczka](plugins.md), moduł zostanie transparentnie zdekompresowany. Informacje o tych załadowanych (i zdekompresowanych) modułach
zostaną przekazane do zgodnego z Multiboot2 jądra w tagach *Moduły* (typ 3).

Szczególnym przypadkiem jest, gdy moduł zaczyna się od bajtów `DSDT`, `GUDT` lub `0xD00DFEED`. W takich przypadkach plik nie
zostanie dodany do tagów *Moduły*, zamiast tego tabela ACPI zostanie poprawiona tak, aby jej wskaźniki DSDT wskazywały na zawartość
tego pliku. Dzięki temu możesz łatwo zastąpić wadliwą tabelę ACPI BIOS-u tabelą dostarczoną przez użytkownika.

### Logo na butach

Możesz również wyświetlić logo na środku ekranu, gdy wybrana jest opcja rozruchu, jeśli umieścisz linię zaczynającą się od
`bootsplash`. Musi być poprzedzona linią `menuentry`.

```
bootsplash [#(kolor tła)] (ścieżka do pliku tga)
```

Kolor tła jest opcjonalny i musi być w notacji HTML rozpoczynającej się od znaku hash, po którym następuje 6 cyfr szesnastkowych,
CCZZNN. Jeśli pierwszy argument nie zaczyna się od `#`, wówczas przyjmuje się argument ścieżki.

Ścieżka musi wskazywać na istniejący plik i musi być absolutną ścieżką UTF-8 na partycji rozruchowej (NIE root). Obraz musi być w
formacie Targa z zakodowaną długością biegu, mapowanym kolorami, tak jak ikony menu. Wymiary mogą być dowolne, aby zmieściły się na
ekranie.

### Obsługa wielu rdzeni

Aby uruchomić jądro na wszystkich rdzeniach procesora jednocześnie, określ dyrektywę `multicore` (tylko jądra 64-bitowe). Musi być
poprzedzona linią `menuentry`.

```
multicore
```

Rozwiązywanie problemów
-----------------------

Jeśli napotkasz jakiekolwiek problemy, po prostu uruchom z flagą `easyboot -vv`. Spowoduje to wykonanie walidacji i wyświetlenie
wyników w trybie verbose w momencie tworzenia obrazu. W przeciwnym razie dodaj `verbose 3` do `easyboot/menu.cfg`, aby uzyskać
szczegółowe komunikaty o czasie rozruchu.

Jeśli widzisz ciąg `PMBR-ERR` w lewym górnym rogu na czerwonym tle, oznacza to, że twój procesor jest bardzo stary i nie obsługuje
64-bitowego trybu długiego lub sektor rozruchowy nie był w stanie uruchomić programu ładującego. Może się to zdarzyć tylko na
komputerach z BIOS-em, nigdy nie może się to zdarzyć na UEFI lub na RaspberryPi.

| Wiadomość                           | Opis                                                                              |
|-------------------------------------|-----------------------------------------------------------------------------------|
| `Booting [X]...`                    | oznacza, że ​​wybrano opcję rozruchu (indeks wpisu menu) X                          |
| `Loading 'X' (Y bytes)...`          | ładowany jest plik X o długości Y                                                 |
| `Parsing kernel...`                 | znaleziono jądro, teraz wykrywamy jego format                                     |
| `Starting X boot...`                | pokazuje, że wykryto program ładujący systemu X                                   |
| `Starting X kernel...`              | pokazuje, że wykryto jądro systemu X                                              |
| `Transfering X control to Y`        | oznacza, że punkt wejścia trybu X pod adresem Y zostanie wkrótce wywołany         |

Jeśli napotkasz jakiekolwiek problemy po wyświetleniu ostatniego komunikatu, oznacza to, że problem wystąpił w procedurze rozruchu
systemu operacyjnego, a nie w programie ładującym **Easyboot**, więc musisz zapoznać się z dokumentacją danego systemu operacyjnego,
aby uzyskać odpowiedź. W przeciwnym razie możesz otworzyć [problem](https://gitlab.com/bztsrc/easyboot/-/issues) w gitlab.

### Multiboot1

Wymagane wtyczki: [grubmb1](../../src/plugins/grubmb1.c)

### Multiboot2

Jest to najbardziej elastyczne rozwiązanie, z obsługą wielu wariantów za pomocą wtyczek:

- ELF64 lub PE32+ z uproszczonym Multiboot2: nie są wymagane żadne wtyczki
- ELF32 z uproszczonym Multiboot2 i 32-bitowym punktem wejścia: [elf32](../../src/plugins/elf32.c)
- a.out (struct exec) z uproszczonym Multiboot2 i 32-bitowym punktem wejścia: [aout](../../src/plugins/aout.c)
- Multiboot2 zgodny z GRUB z 32-bitowym punktem wejścia: [grubmb2](../../src/plugins/grubmb2.c)

Zwróć uwagę na różnicę: [uproszczony Multiboot2](ABI.md) nie wymaga osadzonych znaczników, obsługuje jądra wyższej połowy, czysty
64-bitowy punkt wejścia z parametrami przekazywanymi zgodnie z Multiboot2, SysV i fastcall ABI.

Z drugiej strony [Multiboot2 zgodny z GRUB](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html) wymaga osadzonych
znaczników, nie obsługuje jąder wyższych wersji ani jąder 64-bitowych, punkt wejścia jest zawsze 32-bitowy, a parametry przekazywane
są tylko w `eax`, `ebx`.

### Windows

Nie są wymagane żadne wtyczki, ale musisz skonfigurować [Secure Boot](secureboot.md).

```
menuentry win {
  kernel EFI/Microsoft/BOOT/BOOTMGRW.EFI
}
```

### Linux

Wymagane wtyczki: [linux](../../src/plugins/linux.c), [ext234](../../src/plugins/ext234.c)

Jeśli jądro nie jest umieszczone na partycji rozruchowej, możesz użyć wiersza poleceń, aby określić partycję główną.

```
menuentry linux {
  kernel vmlinuz-linux root=PARTUUID=01020304-0506-0708-0a0b0c0d0e0f1011
}
```

### OpenBSD

Wymagane wtyczki: [obsdboot](../../src/plugins/obsdboot.c), [ufs44](../../src/plugins/ufs44.c)

```
menuentry openbsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot
}
```

OSTRZEŻENIE! Nie używaj wtyczki [elf32](../../src/plugins/elf32.c), jeśli uruchamiasz OpenBSD! Jej `boot` błędnie twierdzi, że jest
ELF z 32-bitowym punktem wejścia SysV ABI, ale w rzeczywistości ma 16-bitowy punkt wejścia w trybie rzeczywistym.

### FreeBSD

Wymagane wtyczki: [fbsdboot](../../src/plugins/fbsdboot.c), [ufs2](../../src/plugins/ufs2.c)

W starszych systemach BIOS określ ładowarkę `boot`.

```
menuentry freebsd {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel boot/boot
}
```

Na komputerach z UEFI użyj `loader.efi` na partycji rozruchowej (nie są wymagane żadne wtyczki).

```
menuentry freebsd {
  kernel boot/loader.efi
}
```

Jeśli twoim głównym systemem plików jest ZFS, skopiuj ten jeden plik (`boot` w BIOS-ie, `loader.efi` w UEFI) do `(indir)` i NIE
określaj partycji głównej.

### FreeDOS

Wymagane wtyczki: [fdos](../../src/plugins/fdos.c)

Przenieś pliki FreeDOS do `(indir)` (FreeDOS użyje partycji rozruchowej jako partycji głównej).

```
menuentry freedos {
  kernel KERNEL.SYS
}
```

Jeżeli rozruch zatrzymuje się po wyświetleniu copyright i `- InitDisk`, oznacza to, że jądro FreeDOS zostało skompilowane bez
obsługi FAT32. Pobierz inne jądro z `f32` w nazwie.

### ReactOS

Wymagane wtyczki: [reactos](../../src/plugins/reactos.c)

```
menuentry reactos {
  kernel FREELDR.SYS
}
```

### MenuetOS

Wymagane wtyczki: [menuet](../../src/plugins/menuet.c)

```
menuentry menuetos {
  kernel KERNEL.MNT
  module CONFIG.MNT
  module RAMDISK.MNT
}
```

Aby wyłączyć automatyczną konfigurację, dodaj `noauto` do wiersza poleceń.

### KolibriOS

Wymagane wtyczki: [kolibri](../../src/plugins/kolibri.c)

```
menuentry kolibrios {
  kernel KERNEL.MNT syspath=/rd/1/ launcher_start=1
  module KOLIBRI.IMG
  module DEVICES.DAT
}
```

Wtyczka działa również na komputerach z UEFI, ale można też użyć `uefi4kos.efi` na partycji rozruchowej (bez konieczności
instalowania wtyczki).

### SerenityOS

Wymagane wtyczki: [grubmb1](../../src/plugins/grubmb1.c)

```
menuentry serenityos {
  kernel boot/Prekernel
  module boot/Kernel
}
```

### Haiku

Wymagane wtyczki: [grubmb1](../../src/plugins/grubmb1.c), [befs](../../src/plugins/befs.c)

```
menuentry haiku {
  partition 01020304-0506-0708-0a0b0c0d0e0f1011
  kernel system/packages/haiku_loader-r1~beta4_hrev56578_59-1-x86_64.hpkg
}
```

Na komputerach z UEFI użyj `haiku_loader.efi` na partycji rozruchowej (nie są wymagane żadne wtyczki).

### OS/Z

```
menuentry osz {
  kernel ibmpc/core
  module ibmpc/initrd
}
```

Nie potrzeba żadnych wtyczek.
