Easyboot met UEFI Secure Boot
=============================

Vermijd dit allereerst wanneer u kunt, want dit is een oplichterij die alleen in naam veilig is (zie CVE-2016-332, CVE-2016-3287,
CVE-2020-14372, CVE-2020-20233, CVE-2020-25632, CVE-2020-25647, CVE-2020-27779, CVE-2020-27749, CVE-2021-3418, CVE-2021-20233,
CVE-2021-20225, CVE-2022-21894, CVE-2022-34301, CVE-2022-34302, CVE-2022-34303, etc).

Als u het om welke reden dan ook niet kunt vermijden, vindt u hier de stappen.

Vereisten
---------

1. Download de nieuwste [shim binair](https://kojipkgs.fedoraproject.org/packages/shim) en pak de bestanden `shimx64.efi` en
   `mmx64.efi` uit de RPM.
2. Installeer **Easyboot** op uw schijf zoals gebruikelijk.

Secure Boot instellen
---------------------

1. Hernoem `EFI\BOOT\BOOTX64.EFI` naar `EFI\BOOT\GRUBX64.EFI` op de ESP-partitie.
2. Kopieer `shimx64.efi` naar de ESP als `EFI\BOOT\BOOTX64.EFI`.
3. Kopieer `mmx64.efi` naar de ESP als `EFI\BOOT\MMX64.EFI`.
4. Start opnieuw op en open het UEFI-configuratiemenu.
5. Controleer of de opstartvolgorde `EFI\BOOT\BOOTX64.EFI` bevat en voeg deze toe als deze ontbreekt.
6. Schakel Secure Boot in en sluit de configuratie af.
7. Shim start MokManager.
8. Selecteer `Enroll hash from disk`.
9. Geef `EFI\BOOT\GRUBX64.EFI` op en voeg het toe aan de MokList.
10. Selecteer `Continue boot`.

Hierna (en bij elke volgende herstart) zou **Easyboot** in de beveiligde opstartmodus moeten starten.
