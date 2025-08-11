Pisanie jąder kompatybilnych z Easyboot
=======================================

[Easyboot](https://gitlab.com/bztsrc/easyboot) obsługuje różne kernele za pomocą [wtyczka](plugins.md). Ale jeśli nie zostanie
znaleziona odpowiednia wtyczka, wraca do plików binarnych ELF64 lub PE32+ z uproszczoną (bez potrzeby osadzania czegokolwiek)
odmianą protokołu Multiboot2.

Jest to dokładnie ten sam protokół, którego używa [Simpleboot](https://gitlab.com/bztsrc/simpleboot). Wszystkie przykładowe jądra
w tym repozytorium muszą również działać z **Easyboot**.

Możesz użyć oryginalnego nagłówka multiboot2.h w repozytorium GRUB lub pliku nagłówkowego C/C++ [easyboot.h](../../easyboot.h), aby
łatwiej było używać typedefów. Format binarny niskiego poziomu jest taki sam, możesz również użyć dowolnych istniejących bibliotek
Multiboot2, nawet w językach innych niż C, takich jak ta biblioteka [Rust](https://github.com/rust-osdev/multiboot2/tree/main/multiboot2/src)
na przykład (uwaga: nie jestem w żaden sposób powiązany z tymi deweloperami, po prostu wyszukałem „Rust Multiboot2” i to był
pierwszy wynik).

[[_TOC_]]

Sekwencja rozruchowa
--------------------

### Bootstrapowanie ładowarka

Na maszynach *BIOS* pierwszy sektor dysku jest ładowany do 0:0x7C00 przez firmware i przekazywana mu jest kontrola. W tym sektorze
**Easyboot** ma [boot_x86.asm](../../src/boot_x86.asm), który jest wystarczająco inteligentny, aby zlokalizować i załadować program
ładujący drugiego etapu, a także skonfigurować dla niego tryb długi.

Na maszynach *UEFI* ten sam plik drugiego etapu, zwany `EFI/BOOT/BOOTX64.EFI`, jest ładowany bezpośrednio przez oprogramowanie
układowe. Źródło tego programu ładującego można znaleźć w [loader_x86.c](../../src/loader_x86.c). To wszystko, **Easyboot** to nie
GRUB ani syslinux, które wymagają dziesiątek plików systemowych na dysku. Tutaj nie potrzeba więcej plików, tylko ten jeden (wtyczki
są opcjonalne, żadna nie jest potrzebna do zapewnienia zgodności z Multiboot2).

Na *Raspberry Pi* ładowarka nazywa się `KERNEL8.IMG` i została skompilowana z [loader_rpi.c](../../src/loader_rpi.c).

### Ładowarka

Ten program ładujący jest bardzo starannie napisany do pracy w wielu konfiguracjach. Ładuje on tabelę partycjonowania GUID z dysku i
szuka „partycji systemowej EFI”. Po znalezieniu szuka pliku konfiguracyjnego `easyboot/menu.cfg` na tej partycji rozruchowej. Po
wybraniu opcji rozruchu i poznaniu nazwy pliku jądra program ładujący lokalizuje go i ładuje.

Następnie automatycznie wykrywa format jądra i jest na tyle inteligentny, aby zinterpretować informacje o sekcji i segmencie
dotyczące tego, gdzie załadować co (w razie potrzeby wykonuje mapowanie pamięci na żądanie). Następnie ustawia odpowiednie
środowisko w zależności od wykrytego protokołu rozruchowego (Multiboot2 / Linux / itd. tryb chroniony lub długi, argumenty ABI
itd.). Po tym, jak stan maszyny jest solidny i dobrze zdefiniowany, jako ostatni krok, ładowarka przechodzi do punktu wejścia jądra.

Stan maszyny
------------

Wszystko, co jest napisane w [specyfikacji Multiboot2](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html) o stanie
maszyny, jest aktualne, z wyjątkiem rejestrów ogólnego przeznaczenia. **Easyboot** przekazuje dwa argumenty do punktu wejścia jądra
zgodnie z SysV ABI i Microsoft fastcall ABI. Pierwszy parametr to magia, drugi to fizyczny adres pamięci, wskazujący na listę tagów
Multiboot Information (w skrócie MBI, patrz poniżej).

Naruszamy również nieco protokół Multiboot2, aby obsługiwać kernele wyższej połowy. Multiboot2 nakazuje, aby pamięć była mapowana
tożsamościowo. Cóż, w przypadku **Easyboot** jest to tylko częściowo prawdą: gwarantujemy tylko, że cała fizyczna pamięć RAM jest z
pewnością mapowana tożsamościowo zgodnie z oczekiwaniami; jednak niektóre regiony powyżej tego (w zależności od nagłówków programu
kernela) mogą być jeszcze dostępne. Nie psuje to normalnych kerneli zgodnych z Multiboot2, które nie powinny uzyskiwać dostępu do
pamięci poza dostępną fizyczną pamięcią RAM.

Twoje jądro jest ładowane dokładnie w ten sam sposób w systemach BIOS i UEFI, a także na RPi, różnice w oprogramowaniu układowym
to po prostu „problem kogoś innego”. Jedyne, co zobaczy twoje jądro, to czy MBI zawiera znacznik tabeli systemowej EFI, czy nie.
Aby uprościć ci życie, **Easyboot** nie generuje również znacznika mapy pamięci EFI (typ 17), dostarcza tylko znacznik
[mapa pamięci](#mapa_pamięci) (typ 6) bez rozróżnienia na wszystkich platformach (również w systemach UEFI, tam mapa pamięci jest
po prostu konwertowana, więc twoje jądro musi radzić sobie tylko z jednym rodzajem znacznika listy pamięci). Stare, przestarzałe
znaczniki są również pomijane i nigdy nie są generowane przez tego menedżera rozruchu.

Jądro działa na poziomie nadzorcy (ring 0 w x86, EL1 w ARM), prawdopodobnie na wszystkich rdzeniach procesora równolegle.

GDT nieokreślone, ale ważne. Stos jest ustawiony w pierwszych 640k i rośnie w dół (ale powinieneś zmienić to tak szybko, jak to
możliwe, na stos, który uważasz za godny). Gdy SMP jest włączone, wszystkie rdzenie mają własne stosy, a identyfikator rdzenia
znajduje się na szczycie stosu (ale możesz również uzyskać identyfikator rdzenia w zwykły sposób specyficzny dla platformy,
używając cpuid / mpidr / itd.).

Powinieneś rozważyć IDT jako nieokreślone; IRQ, NMI i przerwania programowe wyłączone. Fikcyjne programy obsługi wyjątków są
skonfigurowane tak, aby wyświetlać minimalny zrzut i zatrzymywać maszynę. Należy polegać na nich tylko w przypadku raportowania,
czy jądro robi spustoszenie, zanim będziesz w stanie skonfigurować własne IDT i programy obsługi, najlepiej tak szybko, jak to
możliwe. Na ARM vbar_el1 jest skonfigurowany tak, aby wywoływać te same fikcyjne programy obsługi wyjątków (chociaż oczywiście
zrzucają różne rejestry).

Framebuffer jest również ustawiony domyślnie. Możesz zmienić rozdzielczość w konfiguracji, ale jeśli nie podano, framebuffer jest
nadal skonfigurowany.

Ważne jest, aby nigdy nie wracać z jądra. Możesz nadpisać dowolną część ładowarki w pamięci (jak tylko skończysz z tagami MBI),
więc po prostu nie ma dokąd wrócić. "Der Mohr hat seine Schuldigkeit getan, der Mohr kann gehen."

Informacje o rozruchu przekazywane do jądra (MBI)
-------------------------------------------------

Na pierwszy rzut oka nie jest to oczywiste, ale specyfikacja Multiboot2 w rzeczywistości definiuje dwa całkowicie niezależne
zestawy tagów:

- Pierwszy zestaw powinien być wstawiony w kernel zgodny z Multiboot2, nazywany nagłówkiem obrazu systemu operacyjnego Multiboot2
  (sekcja 3.1.2), a zatem *dostarczany przez jądra*. **Easyboot** nie przejmuje się tymi tagami i nie analizuje również kernela
  pod kątem tych tagów. Po prostu nie potrzebujesz żadnych specjalnych magicznych danych osadzonych w pliku kernela z **Easyboot**,
  wystarczą nagłówki ELF i PE.

- Drugi zestaw jest *przekazywany do jądra* dynamicznie podczas rozruchu, **Easyboot** używa tylko tych tagów. Jednak nie generuje
  wszystkiego, co określa Multiboot2 (po prostu pomija stare, przestarzałe lub starsze). Te tagi nazywane są tagami MBI, patrz
  [Boot information](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#Boot-information-format) (sekcja 3.6).

UWAGA: specyfikacja Multiboot2 w tagach MBI jest pełna błędów jak cholera. Poniżej znajdziesz poprawioną wersję, która jest zgodna
z plikiem nagłówkowym multiboot2.h, który znajdziesz w repozytorium źródłowym GRUB.

Pierwszy parametr jądra to magic 0x36d76289 (w `rax`, `rcx` i `rdi`). Możesz zlokalizować znaczniki MBI za pomocą drugiego parametru
(w `rbx`, `rdx` i `rsi`). Na platformie ARM magic jest w `x0`, a address w `x1`. Na RISC-V i MIPS używane są odpowiednio `a0` i `a1`.
Jeśli i kiedy ten program ładujący jest przenoszony do innej architektury, zawsze muszą być używane rejestry określone przez SysV
ABI dla argumentów funkcji. Jeśli na platformie są inne wspólne ABI, które nie kolidują z SysV ABI, wartości powinny być również
duplikowane w rejestrach tych ABI (lub na szczycie stosu).

### Nagłówki

Przekazywany adres jest zawsze wyrównany do 8 bajtów i zaczyna się od nagłówka MBI:

```
        +-------------------+
u32     | total_size        |
u32     | reserved          |
        +-------------------+
```

Następnie następuje seria również 8-bajtowych wyrównanych tagów. Każdy tag zaczyna się od następujących pól nagłówka tagu:

```
        +-------------------+
u32     | type              |
u32     | size              |
        +-------------------+
```

`type` zawiera identyfikator zawartości reszty znacznika. `size` zawiera rozmiar znacznika, wliczając pola nagłówka, ale nie
wliczając wypełnienia. Znaczniki następują po sobie, wypełniane w razie potrzeby, aby każdy znacznik zaczynał się od adresu
wyrównanego do 8 bajtów.

### Terminatora

```
        +-------------------+
u32     | type = 0          |
u32     | size = 8          |
        +-------------------+
```

Tagi zakończone są tagiem typu `0` i rozmiaru `8`.

### Wiersz poleceń rozruchowych

```
        +-------------------+
u32     | type = 1          |
u32     | size              |
u8[n]   | string            |
        +-------------------+
```

`string` zawiera wiersz poleceń określony w wierszu `kernel` *menuentry* (bez ścieżki i nazwy pliku jądra). Wiersz poleceń jest
normalnym ciągiem UTF-8 zakończonym zerem w stylu C.

### Nazwa programu ładującego

```
        +----------------------+
u32     | type = 2             |
u32     | size = 17            |
u8[n]   | string "Easyboot"    |
        +----------------------+
```

`string` zawiera nazwę bootloadera uruchamiającego jądro. Nazwa jest normalnym ciągiem znaków UTF-8 w stylu C zakończonym zerem.

### Moduły

```
        +-------------------+
u32     | type = 3          |
u32     | size              |
u32     | mod_start         |
u32     | mod_end           |
u8[n]   | string            |
        +-------------------+
```

Ten znacznik wskazuje jądru, jaki moduł rozruchowy został załadowany wraz z obrazem jądra i gdzie można go znaleźć. `mod_start` i
`mod_end` zawierają początkowy i końcowy adres fizyczny samego modułu rozruchowego. Nigdy nie otrzymasz skompresowanego bufora gzip,
ponieważ **Easyboot** transparentnie je dekompresuje (a jeśli dostarczysz wtyczkę, działa również z innymi skompresowanymi danymi
niż gzip). Pole `string` dostarcza dowolny ciąg, który ma być skojarzony z tym konkretnym modułem rozruchowym; jest to normalny
ciąg UTF-8 zakończony zerem w stylu C. Określony w wierszu `module` *menuentry*, a jego dokładne użycie jest specyficzne dla
systemu operacyjnego. W przeciwieństwie do znacznika wiersza poleceń rozruchowych, znaczniki modułu *również zawierają* ścieżkę i
nazwę pliku modułu.

Jeden tag pojawia się na moduł. Ten typ tagu może pojawić się wiele razy. Jeśli początkowy ramdisk został załadowany wraz z
kernelem, pojawi się on jako pierwszy moduł.

Istnieje szczególny przypadek, gdy plik jest tabelą DSDT ACPI, FDT (dtb) lub blobem GUDT, wówczas nie pojawi się on jako moduł.
Zamiast tego ACPI stary RSDP (typ 14) lub ACPI nowy RSDP (typ 15) zostanie załatany, a ich DSDT zostanie zastąpione zawartością
tego pliku.

### Mapa pamięci

Ten znacznik zapewnia mapę pamięci.

```
        +-------------------+
u32     | type = 6          |
u32     | size              |
u32     | entry_size = 24   |
u32     | entry_version = 0 |
varies  | entries           |
        +-------------------+
```

`size` zawiera rozmiar wszystkich wpisów, włączając w to samo pole. `entry_size` jest zawsze 24. `entry_version` jest ustawione
na `0`. Każdy wpis ma następującą strukturę:

```
        +-------------------+
u64     | base_addr         |
u64     | length            |
u32     | type              |
u32     | reserved          |
        +-------------------+
```

`base_addr` to początkowy adres fizyczny. `length` to rozmiar obszaru pamięci w bajtach. `type` to odmiana zakresu adresów, gdzie
wartość `1` oznacza dostępną pamięć RAM, wartość `3` oznacza użyteczną pamięć przechowującą informacje ACPI, wartość `4` oznacza
zarezerwowaną pamięć, która musi zostać zachowana podczas hibernacji, wartość `5` oznacza pamięć, która jest zajmowana przez
uszkodzone moduły RAM, a wszystkie inne wartości obecnie wskazują na zarezerwowany obszar. `reserved` jest ustawione na `0` podczas
rozruchu BIOS-u.

Po wygenerowaniu MBI na komputerze UEFI różne wpisy mapy pamięci EFI są przechowywane jako typ `1` (dostępna pamięć RAM) lub `2`
(zarezerwowana pamięć RAM), a jeśli zajdzie taka potrzeba, oryginalny typ pamięci EFI jest umieszczany w polu `reserved`.

Dostarczona mapa gwarantuje, że wymieni całą standardową pamięć RAM, która powinna być dostępna do normalnego użytku, i zawsze jest
uporządkowana rosnąco według `base_addr`. Ten dostępny typ pamięci RAM obejmuje jednak regiony zajmowane przez jądro, mbi, segmenty
i moduły. Jądro musi uważać, aby nie nadpisać tych regionów (**Easyboot** mógłby łatwo wykluczyć te regiony, ale to zepsułoby
zgodność z Multiboot2).

### Informacje o buforze ramki

```
        +----------------------------------+
u32     | type = 8                         |
u32     | size = 38                        |
u64     | framebuffer_addr                 |
u32     | framebuffer_pitch                |
u32     | framebuffer_width                |
u32     | framebuffer_height               |
u8      | framebuffer_bpp                  |
u8      | framebuffer_type = 1             |
u16     | reserved                         |
u8      | framebuffer_red_field_position   |
u8      | framebuffer_red_mask_size        |
u8      | framebuffer_green_field_position |
u8      | framebuffer_green_mask_size      |
u8      | framebuffer_blue_field_position  |
u8      | framebuffer_blue_mask_size       |
        +----------------------------------+
```

Pole `framebuffer_addr` zawiera fizyczny adres bufora ramki. Pole `framebuffer_pitch` zawiera długość jednego wiersza w bajtach.
Pola `framebuffer_width`, `framebuffer_height` zawierają wymiary bufora ramki w pikselach. Pole `framebuffer_bpp` zawiera liczbę
bitów na piksel. `framebuffer_type` jest zawsze ustawione na 1, a `reserved` zawsze zawiera 0 w obecnej wersji specyfikacji i musi
być ignorowane przez obraz systemu operacyjnego. Pozostałe pola opisują spakowany format pikseli, pozycję i rozmiar kanałów w
bitach. Możesz użyć wyrażenia `((~(0xffffffff << size)) << position) & 0xffffffff`, aby uzyskać maskę kanału podobną do UEFI GOP.

### Wskaźnik tabeli systemowej EFI 64-bit

Ten tag istnieje tylko wtedy, gdy **Easyboot** jest uruchomiony na maszynie UEFI. Na maszynie BIOS ten tag nigdy nie został
wygenerowany.

```
        +-------------------+
u32     | type = 12         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

Ten tag zawiera wskaźnik do EFI system table.

### Wskaźnik uchwytu obrazu EFI 64-bit

Ten tag istnieje tylko wtedy, gdy **Easyboot** jest uruchomiony na maszynie UEFI. Na maszynie BIOS ten tag nigdy nie został
wygenerowany.

```
        +-------------------+
u32     | type = 20         |
u32     | size = 16         |
u64     | pointer           |
        +-------------------+
```

Ten tag zawiera wskaźnik do EFI image handle. Zazwyczaj jest to uchwyt obrazu ładowarki.

### Tabele SMBIOS

```
        +-------------------+
u32     | type = 13         |
u32     | size              |
u8      | major             |
u8      | minor             |
u8[6]   | reserved          |
        | tabele smbios     |
        +-------------------+
```

Ten tag zawiera kopię tabel SMBIOS oraz ich wersję.

### ACPI stary RSDP

```
        +-------------------+
u32     | type = 14         |
u32     | size              |
        | kopia RSDPv1      |
        +-------------------+
```

Ten znacznik zawiera kopię protokołu RSDP zdefiniowanego w specyfikacji ACPI 1.0. (Z adresem 32-bitowym.)

### ACPI nowy RSDP

```
        +-------------------+
u32     | type = 15         |
u32     | size              |
        | kopia RSDPv2      |
        +-------------------+
```

Ten znacznik zawiera kopię protokołu RSDP zdefiniowanego w specyfikacji ACPI 2.0 lub nowszej. (Prawdopodobnie z adresem 64-bitowym.)

Te (typ 14 i 15) wskazują na tabelę `RSDT` lub `XSDT` ze wskaźnikiem do tabeli `FACP`, która z kolei zawiera dwa wskaźniki do
tabeli `DSDT`, która opisuje maszynę. **Easyboot** fałszuje te tabele na maszynach, które w inny sposób nie obsługują ACPI. Ponadto,
jeśli dostarczysz tabelę DSDT, blob FDT (dtb) lub GUDT jako moduł, **Easyboot** załata wskaźniki, aby wskazywały na tę dostarczoną
przez użytkownika tabelę. Aby przeanalizować te tabele, możesz użyć mojej niezależnej biblioteki nagłówkowej
[hwdet](https://gitlab.com/bztsrc/hwdet) (lub rozdętej [apcica](https://github.com/acpica/acpica) i [libfdt](https://github.com/dgibson/dtc)).

Tagi specyficzne dla jądra
--------------------------

Znaczniki z `type` większym lub równym 256 nie są częścią specyfikacji Multiboot2, mimo to są dostarczane przez **Easyboot**. Mogą
być dodawane przez opcjonalne [wtyczki](plugins.md) do listy, jeśli i kiedy jądro ich potrzebuje.

### EDID

```
        +-------------------+
u32     | type = 256        |
u32     | size              |
        | kopia EDID        |
        +-------------------+
```

Ten tag zawiera kopię listy obsługiwanych rozdzielczości monitora zgodnie ze specyfikacją EDID.

### SMP

```
        +-------------------+
u32     | type = 257        |
u32     | size              |
u32     | numcores          |
u32     | running           |
u32     | bspid             |
        +-------------------+
```

Ten znacznik istnieje, jeśli podano dyrektywę `multicore`. `numcores` zawiera liczbę rdzeni procesora w systemie, `running` to
liczba rdzeni, które pomyślnie zainicjowały i uruchomiły to samo jądro równolegle. `bspid` zawiera identyfikator rdzenia BSP (na
identyfikatorze x86 lAPIC), dzięki czemu jądra mogą rozróżniać punkty dostępowe i uruchamiać na nich inny kod. Wszystkie punkty
dostępowe mają własny stos, a na szczycie stosu będzie znajdował się identyfikator bieżącego rdzenia.

### Identyfikatory partycji

```
        +-------------------+
u32     | type = 258        |
u32     | size = 24 / 40    |
u128    | bootuuid          |
u128    | rootuuid          |
        +-------------------+
```

Ten tag zawiera pola unikatowego identyfikatora w GPT partycji rozruchowej i głównej. Jeśli rozruch nie używa tabeli partycjonowania
GUID, wówczas `bootuuid` jest generowany jako `54524150-(kod urządzenia)-(numer partycji)-616F6F7400000000`.

Układ pamięci
-------------

### Maszyny BIOS

| Początek | Koniec  | Opis                                                            |
|---------:|--------:|-----------------------------------------------------------------|
|      0x0 |   0x400 | tabela wektorów przerwań (użyteczna, tryb rzeczywisty IDT)      |
|    0x400 |   0x4FF | obszar danych BIOS-u (użyteczny)                                |
|    0x4FF |   0x500 | kod dysku rozruchowego BIOS-u (prawdopodobnie 0x80, użyteczny)  |
|    0x500 |   0x5A0 | dane synchronizacyjne dla SMP (użyteczne)                       |
|    0x5A0 |  0x1000 | stos obsługi wyjątków (można ich używać po skonfigurowaniu IDT) |
|   0x1000 |  0x8000 | tabele stronicowania (można ich używać po skonfigurowaniu tabel stronicowania) |
|   0x8000 | 0x20000 | kod i dane ładowarki (można ich używać po skonfigurowaniu IDT)  |
|  0x20000 | 0x40000 | konfiguracja + tagi (można ich używać po przeanalizowaniu MBI)  |
|  0x40000 | 0x90000 | identyfikatory wtyczek; od góry do dołu: stos jądra             |
|  0x90000 | 0x9A000 | tylko jądro Linux: zero page + cmdline                          |
|  0x9A000 | 0xA0000 | rozszerzony obszar danych BIOS-u (lepiej nie dotykać)           |
|  0xA0000 | 0xFFFFF | pamięć VRAM i BIOS ROM (nieużyteczne)                           |
| 0x100000 |       x | segmenty jądra, po których następują moduły, każda strona wyrównana |

### Maszyny UEFI

Nikt nie wie. UEFI przydziela pamięć, jak mu się podoba. Spodziewaj się wszystkiego. Wszystkie obszary będą z pewnością wymienione
na mapie pamięci jako type = 1 (`MULTIBOOT_MEMORY_AVAILABLE`) i reserved = 2 (`EfiLoaderData`), jednak nie jest to wyłączna cecha,
inne rodzaje pamięci również mogą być wymienione w ten sposób (na przykład sekcja bss menedżera rozruchu).

### Raspberry Pi

| Początek | Koniec  | Opis                                                                |
|---------:|--------:|---------------------------------------------------------------------|
|      0x0 |   0x500 | zarezerwowane przez oprogramowanie układowe (lepiej nie ruszać)     |
|    0x500 |   0x5A0 | dane synchronizacyjne dla SMP (użyteczne)                           |
|    0x5A0 |  0x1000 | stos obsługi wyjątków (można ich używać po skonfigurowaniu VBAR)    |
|   0x1000 |  0x9000 | tabele stronicowania (można ich używać po skonfigurowaniu tabel stronicowania) |
|   0x9000 | 0x20000 | kod i dane ładowarki (można ich używać po skonfigurowaniu VBAR)     |
|  0x20000 | 0x40000 | konfiguracja + tagi (można ich używać po przeanalizowaniu MBI)      |
|  0x40000 | 0x80000 | oprogramowanie układowe dostarczone przez FDT (dtb); od góry do dołu: stos jądra |
| 0x100000 |       x | segmenty jądra, po których następują moduły, każda strona wyrównana |

Pierwsze kilka bajtów jest zarezerwowanych dla [armstub](https://github.com/raspberrypi/tools/blob/master/armstubs/armstub8.S).
Uruchomiono tylko rdzeń 0, więc aby uruchomić procesory aplikacji, zapisz adres funkcji do 0xE0 (rdzeń 1), 0xE8 (rdzeń 2), 0xF0
(rdzeń 3), które to adresy znajdują się w tym obszarze. Nie ma to znaczenia, gdy używana jest dyrektywa `multicore`, wtedy wszystkie
rdzenie wykonają jądro.

Chociaż natywnie nie jest obsługiwany na RPi, nadal otrzymujesz tag ACPI stary RSDP (typ 14) z fałszywymi tabelami. Tabela `APIC`
jest używana do komunikowania liczby dostępnych rdzeni procesora do jądra. Adres funkcji startowej jest przechowywany w polu RSD
PTR-> RSDT -> APIC -> cpu\[x].apic_id (a identyfikator rdzenia w cpu\[x].acpi_id, gdzie BSP to zawsze cpu\[0].acpi_id = 0 i
cpu\[0].apic_id = 0xD8. Uważaj, „acpi” i „apic” wyglądają dość podobnie).

Jeśli prawidłowy blob FDT zostanie przekazany przez firmware lub jeśli jeden z modułów jest plikiem .dtb, .gud lub .aml, dodawana
jest również tabela FADT (z magicznym `FACP`). W tej tabeli wskaźnik DSDT (32-bitowy, przy przesunięciu 40) wskazuje na dostarczony
spłaszczony blob drzewa urządzeń.

Chociaż oprogramowanie układowe nie zapewnia funkcji mapy pamięci, otrzymasz również znacznik Mapa pamięci (typ 6), zawierający
wykrytą pamięć RAM i region MMIO. Możesz go użyć do wykrycia adresu bazowego MMIO, który jest inny w RPi3 i RPi4.
