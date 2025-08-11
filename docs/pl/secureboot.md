Easyboot z UEFI Secure Boot
===========================

Przede wszystkim unikaj tego, kiedy tylko możesz, ponieważ jest to oszustwo, bezpieczne tylko pod swoją nazwą (patrz CVE-2016-332,
CVE-2016-3287, CVE-2020-14372, CVE-2020-20233, CVE-2020-25632, CVE-2020-25647, CVE-2020-27779, CVE-2020-27749, CVE-2021-3418,
CVE-2021-20233, CVE-2021-20225, CVE-2022-21894, CVE-2022-34301, CVE-2022-34302, CVE-2022-34303 itd.).

Jeśli z jakiegoś powodu nie możesz tego uniknąć, oto co możesz zrobić.

Wymagania wstępne
-----------------

1. Pobierz najnowszy [plik binarny shim](https://kojipkgs.fedoraproject.org/packages/shim) i wypakuj pliki `shimx64.efi` oraz
   `mmx64.efi` z RPM.
2. Zainstaluj **Easyboot** na swoim dysku w zwykły sposób.

Konfigurowanie Secure Boot
--------------------------

1. Zmień nazwę `EFI\BOOT\BOOTX64.EFI` na `EFI\BOOT\GRUBX64.EFI` na partycji ESP.
2. Skopiuj `shimx64.efi` do ESP jako `EFI\BOOT\BOOTX64.EFI`.
3. Skopiuj `mmx64.efi` do ESP jako `EFI\BOOT\MMX64.EFI`.
4. Uruchom ponownie i wejdź do menu konfiguracji UEFI.
5. Sprawdź, czy w kolejności rozruchowej znajduje się `EFI\BOOT\BOOTX64.EFI`, dodaj go, jeśli go brakuje.
6. Włącz Bezpieczny rozruch i zakończ konfigurację.
7. Shim uruchomi MokManager.
8. Wybierać `Enroll hash from disk`.
9. Określ `EFI\BOOT\GRUBX64.EFI` i dodaj go do MokList.
10. Wybierać `Continue boot`.

Następnie (i przy każdym kolejnym ponownym uruchomieniu) **Easyboot** powinien uruchomić się w trybie Secure Boot.
