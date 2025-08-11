Easyboot com UEFI Inicialização Segura
======================================

Em primeiro lugar, evite sempre que puder, pois trata-se de um golpe, seguro apenas em seu nome (ver CVE-2016-332, CVE-2016-3287,
CVE-2020-14372, CVE-2020-20233, CVE-2020-25632, CVE-2020-25647, CVE-2020-27779, CVE-2020-27749, CVE-2021-3418, CVE-2021-20233,
CVE-2021-20225, CVE-2022-21894, CVE-2022-34301, CVE-2022-34302, CVE-2022-34303, etc).

Se não puder evitar por qualquer motivo, aqui estão os passos.

Pré-requisitos
--------------

1. Descarregue o [binário shim](https://kojipkgs.fedoraproject.org/packages/shim) mais recente e extraia os ficheiros `shimx64.efi`
   e `mmx64.efi` do RPM.
2. Instale o **Easyboot** no seu disco como habitualmente.

Configurando a inicialização segura
-----------------------------------

1. Renomeie `EFI\BOOT\BOOTX64.EFI` para `EFI\BOOT\GRUBX64.EFI` na partição ESP.
2. Copie `shimx64. efi` para o ESP como `EFI\BOOT\BOOTX64. EFI`.
3. Copie `mmx64. efi` para o ESP como `EFI\BOOT\MMX64. EFI`.
4. Reinicie e entre no menu de configuração UEFI.
5. Verifique se a ordem de arranque tem `EFI\BOOT\BOOTX64.EFI` e adicione-o se estiver em falta.
6. Ative a Inicialização Segura e saia da configuração.
7. Shim iniciará o MokManager.
8. Selecionar `Enroll hash from disk`.
9. Especifique `EFI\BOOT\GRUBX64.EFI` e adicione-o à MokList.
10. Selecionar `Continue boot`.

Depois disso (e em todas as reinicializações seguintes), o **Easyboot** deverá iniciar no modo de arranque seguro.
