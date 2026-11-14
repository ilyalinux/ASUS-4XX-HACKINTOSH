# hackintosh-asus-4xx

[English](README.md) | [Русский](README.ru.md)

---

### Настройка OpenCore для материнских плат ASUS 4XX

Руководство было написано после установки macOS 26 Tahoe на ASUS PRIME H470-PLUS с i3-10100F, актуально в том числе для H470, B460, Z490 и прочих, но в зависимости от вашей конфигурации могут быть иные проблемы или способы их решения.  
В этом репозитории собраны проверенные кексты, таблицы ACPI и настройки `config.plist`, решающие возможные проблемы.

---

### 1. Баг памяти MAT в некоторых версиях BIOS ASUS
* Ядро macOS намертво зависает на строчке `#[EB.LD.LKC|R.1]<"BootKernelExtensions.kc">`. В определенных версиях BIOS ASUS повреждена таблица Memory Attributes Table, дефолтные настройки от Dortania приводят к зависанию. В секции `Booter --> Quirks` необходимо выставить:
  ```xml
  <key>EnableWriteUnprotector</key>
  <true/>
  <key>RebuildAppleMemoryMap</key>
  <false/>
  <key>SyncRuntimePermissions</key>
  <false/>
  <key>SetupVirtualMap</key>
  <false/>
  ```

---

### 2. Падение BIOS в safe mode при выключении
* При выключении или перезагрузке материнская плата проходит POST в safe mode. Драйвер AppleRTC перезаписывает контрольные суммы в CMOS. BIOS воспринимает это как сбой питания/батарейки. В секции `Kernel --> Quirks` включить:
  ```xml
  <key>DisableRtcChecksum</key>
  <true/>
  ```

---

### 3. Не рабочий RealtekRTL8111.kext v3
* Это актуально скорее только под ASUS PRIME H470. Последняя версия RealtekRTL8111.kext v3.0.0 зависает на согласовании линка на 15-й ревизии чипа без AppleVTD. Помогает откат на версию RealtekRTL8111.kext v2.4.2 (включена в каталог Kexts).

---

### 4. Нет звука в macOS 26 Tahoe (AppleHDA)
* Apple полностью удалила AppleHDA.kext из состава macOS Tahoe. Кекст `AppleALC` не может работать без базового драйвера. Необходимо добавить кекст **`SimpleHDA.kext`** (автор laobamac) в `EFI/OC/Kexts/`. Порядок в `Kernel --> Add` строго: `Lilu.kext` --> `AppleALC.kext` --> `SimpleHDA.kext`.
  1. Включить `Kernel --> Quirks --> DisableLinkeditJettison = True`
  2. Для 3-разъемных плат (например, на ALC887-VD) использовать `alcid=11` в boot-args и убрать принудительный layout-id из DeviceProperties.

---

### Какие кексты лежат в каталоге Kexts и зачем они нужны

* **`Lilu.kext`** -- в списке `Kernel --> Add` он **всегда** должен быть на 1-м месте. Предоставляет API для перехвата и модификации функций ядра на лету, обязателен для всех остальных кекстов.
* **`VirtualSMC.kext`** -- эмуляция аппаратного чипа Apple SMC. Обходит опрос ключей подлинности и датчиков.
* **`AppleALC.kext`** -- драйвер звука для встроенных кодеков Realtek (в том числе ALC887-VD). Динамически подменяет конфигурации аудио и разъемов платы под нужный профиль.
* **`SimpleHDA.kext`** -- в конфиге должен стоять строго после AppleALC.kext. Только для macOS 26 Tahoe: Apple полностью вырезала системный AppleHDA.kext, кексту AppleALC стало нечего патчить. SimpleHDA на лету возвращает стек AppleHDA.
* **`RealtekRTL8111.kext` (версия 2.4.2)** -- сетевая карта проводного интернета. Положена именно версия **2.4.2**, так как свежая версия 3.0.0 зависает на этапе согласования скорости на чипах ревизии 15 без AppleVTD.
* **`USBToolBox.kext` + `UTBDefault.kext`** -- связка для работы портов USB. Позволяет сразу запустить клавиатуру, мышь и флешки на любых портах материнской платы без необходимости вручную маппить порты.
* **`RestrictEvents.kext`** -- универсальный патчер системных ограничений от Acidanthera. Необходим для Tahoe, чтобы ОС не блокировала загрузку на неподдерживаемых процессорах и SMBIOS (через аргумент `revpatch=sbvmm`), а также блокирует лишние системные предупреждения.

---

### Таблицы в каталоге ACPI (.aml)

* **`SSDT-AWAC.aml`** -- исправляет системные часы реального времени (RTC). На чипсетах Intel 400-й серии по умолчанию включен таймер AWAC, несовместимый с macOS. Этот патч отключает AWAC и активирует классический RTC.
* **`SSDT-PLUG-DRTNIA.aml`** -- включает родной CPU Power Management (`plugin-type=1`), благодаря чему процессор правильно сбрасывает частоты в простое.
* **`SSDT-EC-USBX-DESKTOP.aml`** -- создает фиктивное устройство встроенного контроллера и задает параметры тока питания USB.

---

### Обязательные аргументы загрузки (boot-args)
В `NVRAM --> Add --> 7C436110-AB2A-4BBB-A880-FE41995C9F82 --> boot-args`:
```text
-lilubetaall -vsmcbeta -revbeta revpatch=sbvmm alcid=11
```
* `-lilubetaall -vsmcbeta -revbeta` -- принудительно разблокируют работу Lilu и плагинов на новом ядре Darwin 25/26 (macOS Tahoe).
* `revpatch=sbvmm` -- маскирует загрузку для установщика и апдейтов через RestrictEvents.
* `alcid=11` -- задает корректный лейаут 3-разъемного аудиокодека Realtek ALC887-VD платы ASUS.
