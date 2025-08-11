Easyboot con UEFI Secure Boot
=============================

Innanzitutto, evitalo il più possibile, perché si tratta di una truffa, sicura solo nel suo nome (vedi CVE-2016-332, CVE-2016-3287,
CVE-2020-14372, CVE-2020-20233, CVE-2020-25632, CVE-2020-25647, CVE-2020-27779, CVE-2020-27749, CVE-2021-3418, CVE-2021-20233,
CVE-2021-20225, CVE-2022-21894, CVE-2022-34301, CVE-2022-34302, CVE-2022-34303, ecc.).

Se per qualsiasi motivo non puoi evitarlo, ecco i passaggi da seguire.

Prerequisiti
------------

1. Scarica l'ultimo [binario shim](https://kojipkgs.fedoraproject.org/packages/shim) ed estrai i file `shimx64.efi` e `mmx64.efi`
   dall'RPM.
2. Installa **Easyboot** sul tuo disco come di consueto.

Impostazione dell'Secure Boot
-----------------------------

1. Rinominare `EFI\BOOT\BOOTX64.EFI` in `EFI\BOOT\GRUBX64.EFI` sulla partizione ESP.
2. Copiare `shimx64.efi` nell'ESP come `EFI\BOOT\BOOTX64.EFI`.
3. Copiare `mmx64.efi` nell'ESP come `EFI\BOOT\MMX64.EFI`.
4. Riavviare e accedere al menu di configurazione UEFI.
5. Controllare se l'ordine di avvio ha `EFI\BOOT\BOOTX64.EFI`, aggiungerlo se manca.
6. Abilitare l'avvio protetto e uscire dalla configurazione.
7. Shim avvierà MokManager.
8. Selezionare `Enroll hash from disk`.
9. Specificare `EFI\BOOT\GRUBX64.EFI` e aggiungerlo a MokList.
10. Selezionare `Continue boot`.

Dopo questo (e a ogni riavvio successivo) **Easyboot** dovrebbe avviarsi in modalità Secure Boot.


