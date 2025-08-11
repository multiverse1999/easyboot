Wsparcie Stivale w Easyboot
===========================

To się nie wydarzy, nigdy. Ten protokół rozruchowy ma kilka bardzo poważnych złych wyborów projektowych i jest ogromnym ryzykiem
bezpieczeństwa.

Po pierwsze, jądra stivale mają nagłówek ELF, ale jakoś powinieneś wiedzieć, że nagłówek nie jest prawidłowy; nic, absolutnie nic w
nagłówku nie sugeruje, że nie jest to prawidłowe jądro SysV ABI. W Multiboot muszą być jakieś magiczne bajty na początku pliku, aby
można było wykryć protokół; nie ma niczego takiego w stivale / stivale2. (Zakotwiczenie ci nie pomoże, ponieważ może wystąpić
*w dowolnym miejscu* w pliku, więc tak naprawdę musisz *przeszukać cały plik*, aby upewnić się, że nie jest zgodny ze stivale.)

Po drugie, używa sekcji, które zgodnie ze specyfikacją ELF (patrz [strona 46](https://www.sco.com/developers/devspecs/gabi41.pdf))
są opcjonalne i żaden loader nie powinien się nimi przejmować. Loadery używają widoku wykonania, a nie widoku łączenia.
Implementacja analizy sekcji tylko z powodu tego jednego protokołu jest szalonym narzutem dla każdego loadera, gdzie zasoby
systemowe są zwykle już ograniczone.

Po trzecie, nagłówki sekcji znajdują się na końcu pliku. Oznacza to, że aby wykryć stivale, musisz załadować początek pliku,
przeanalizować nagłówki ELF, następnie załadować koniec pliku i przeanalizować nagłówki sekcji, a następnie załadować gdzieś ze
środka pliku, aby uzyskać rzeczywistą listę tagów. To najgorsze możliwe rozwiązanie. I znowu, nie ma absolutnie nic, co
wskazywałoby, że ładowarka powinna to zrobić, więc musisz to zrobić dla wszystkich jąder, aby dowiedzieć się, że jądro nie używa
stivale. Spowalnia to również wykrywanie *wszystkich innych* protokołów rozruchowych, co jest niedopuszczalne.

Lista tagów jest aktywnie sprawdzana przez procesory aplikacji, a jądro może wywołać kod bootloadera w dowolnym momencie, co
oznacza, że ​​po prostu nie można odzyskać pamięci bootloadera, w przeciwnym razie awaria jest gwarantowana. Jest to sprzeczne z
filozofią **Easyboot**.

Najgorsze jest to, że protokół oczekuje, że bootloadery wstrzykną kod do dowolnego jądra zgodnego ze stivale, który następnie
zostanie wykonany na najwyższym możliwym poziomie uprawnień. Tak, co mogłoby pójść nie tak, prawda?

Ponieważ odmawiam oddawania słabej jakości kodu z moich rąk, nie będzie żadnego wsparcia stivale w **Easyboot**. A jeśli posłuchasz
mojej rady, żaden hobbystyczny deweloper OS nie powinien nigdy używać go dla własnego dobra.
