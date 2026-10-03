# AIHANDOFF — SZZT KS8223 custom kernel (msm8909)

Дата: 2026-10-03. Язык общения: русский. Репо: `Ssss4562/android_kernel_szzt_ks8223`.
Рабочая ветка: та, что была активна (коммиты Ssss4562, HEAD `cfbdf2d` + наши).

## Железо и сборка
- SZZT KS8223 POS, MSM8909 + PM8909, 1GB, QRD SKUA (`board-id 0x1000b/0`, `hw_plat=11`).
  Дисплей Himax hx9483e 720x1280 (MDP3), тач Goodix GT5688 (gt1x, I2C 0x5d, шина 78b9000).
- Конфиг: `arch/arm/configs/msm8909-1gb_defconfig`. Сборка ТОЛЬКО так:
  ```
  export ARCH=arm
  export CROSS_COMPILE=~/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabi/bin/arm-linux-gnueabi-
  make HOSTCFLAGS="-fcommon" CC="${CROSS_COMPILE}gcc" msm8909-1gb_defconfig
  make HOSTCFLAGS="-fcommon" CC="${CROSS_COMPILE}gcc" -j$(nproc) zImage dtbs
  ```
  (`HOSTCFLAGS="-fcommon"` обязателен — иначе падает на новых host-gcc.)
- DT: QCDT v3 собирается из 2 dtb (`msm8909-1gb-qrd-skua.dtb` + `msm8909-qrd-skua.dtb`)
  через `~/qcdt-builder` (`python3 -m qcdt_builder build A B --output qcdt.img`,
  page 2048, итог 8 entries, 313344 байт). `stock_dt.img` в стоковом boot идентичен
  по таблице.
- Boot/recovery образы: ручная упаковка v0 (`base 0x80000000`, `koff 0x8000`,
  `roff 0x1000000`, `tags 0x100`, `page 2048`, cmdline стоковый). Готовые лежат в
  `/home/user/kerneltests/` (`boot_new.img`, `twrp_ks8223.img`, `boot_stock.img`,
  `zImage_fresh`, `qcdt_fresh.img`, dtb, ramdisk, dmesg-логи, README.txt).
  Recovery раздел 32МБ — всё влезает. Тест только через `fastboot boot`, не шить вслепую.
- Стоковая прошивка: `~/Downloads/azurpos/` (boot/recovery/system.img raw ext4, rawprogram.xml).
- TWRP-порт: ramdisk из `~/twrp-3.7.0_9-0-m1.img` (чужой MSM8909) + наше ядро/DT,
  в `recovery.fstab` `soc.0` → `bootdevice`. Тема m1 1080x1920 (на нашей 720x1280
  масштабируется). Тач/экран/разделы в TWRP работают.
- adb на кассе капризный (плохой кабель!): добавлен `/etc/udev/rules.d/51-ks8223.rules`
  (05c6/18d1, MODE 0666). Прямой `adb reboot` иногда не срабатывает — перезапуск
  через рекавери. EDL 9008 лечится долгим Power.

## Что УЖЕ работает (проверено на железе, оставить как есть)
1. **Тач**: `Goodix GT5688 720x1280` (`getevent -p`). Фикс = `# CONFIG_GTP_DRIVER_SEND_CFG
   is not set` в `msm8909-1gb_defconfig` (в `msm8909-1gb-perf_defconfig` уже было):
   драйвер читает конфиг из чипа, иначе X_MAX=Y_MAX=0. (`int cfg_len` в gt1x уже int,
   править нечего.) Фантомный `focaltech@38` (NACK 0x38) остаётся `disabled`.
2. **Батарея**: `ACC_IBAT_ROWS 4 → 6` в `include/linux/batterydata-lib.h` — профиль
   `Easlink5200` имеет 6-строчный `ibat-acc-lut`, драйвер падал `-22`, BMS не
   стартовал. Теперь `bms` есть, `0%→49%`, `voltage_now` растёт, `Charging`.
   (В стоке BMS выключен `disable-bms`, кажет заглушки — так задумано вендором.)
3. **USB**: кастомный DT форсит peripheral `0x01` без GPIO (adb/mtp), сток `0x03` OTG.
   НЕ ТРОГАТЬ — патч пользователя осознанный (защита кассы).

## Что ОТКАЧЕНО в этом коммите (удалено из дерева!)
По просьбе пользователя откачены принтерные/SDK-драйверы и отладка. Восстановить можно
по описанию ниже (исходники удалены, смыслы сохранены):
- `drivers/misc/kingsee_gpio_misc.c` (new, удалён) + `Kconfig/Makefile` записи +
  `CONFIG_KINGSEE_GPIO_MISC=y` + DT-проперти `qcom,FN_3V3_PWR`. Работал, проверен:
  `/dev/k21_dev` (misc), ioctl `0x6D00` reset-импульс (gpio15: 1, sleep 500мс, 0),
  `0x6D01` GEN_3V3:=arg (gpio77), `0x6D02` чтение FN_3V3 (put_user int),
  `0x6D03` FN_3V3:=arg. GPIO: `gen_3v3`=TLMM 77, `k21_reset`=TLMM 15 (DT phandle
  `0x98`), `FN_3V3`=PMIC gpio1 (глобальный 907 = база pm8909 906 + 1, проверено
  через `/sys/kernel/debug/gpio`: `907 FN_3V3_PWR hi`, `926 K21_RST`, `988 GEN_3V3_PWR`).
  Принты как в стоке (`k21_dev_ioctl ...`). Совпадает с дизассемблером стокового ядра.
- `drivers/misc/szzt_printer.c` (new, удалён) + нода `szzt-printer` в обоих DTS +
  `CONFIG_SZZT_PRINTER=y`. Заглушка `/dev/printer` принимала `0xE0/0x7000/0x7001`.
  ВАЖНО: живой тест показал что asdkserver шлёт `ioctl 0x7001` с УКАЗАТЕЛЕМ и ждёт
  запись `ready=1` (иначе через ~80с lockdown, см. ниже). Версия с `put_user(1)`
  была написана, но НЕ СОБРАНА/НЕ ПРОТЕСТИРОВАНА (юзер остановил).
- Ловушки: `drivers/tty/sysrq.c` + `kernel/sys.c` (лог `sysrq-trigger`/`reboot` с
  `comm/pid`) — откачены.
- Оставлено: `mdp3.c` эксперимент (закомментирован вызов `mdp3_continuous_splash_on`
  в ветке `lk continuous splash, but kerenl not` — при включённом cont-splash мёртвый
  код, безвреден; если мешает — откатить одной правкой).

## Открытые проблемы (по приоритету)
1. **Самопроизвольная смерть ~через 200с после загрузки** (главное!). Поймано ловушкой:
   `sysrq-trigger: cmd 'u' by asdkserver` → Emergency Remount R/O → reboot, причина
   PMIC всегда `PS_HOLD (MSM controlled shutdown)`, в ядре НИКАКИХ ошибок (паник/
   вотчдогов/термала нет). На стоке 10-минутный вахт — жив. asdkserver держит
   `/dev/ttyHSL1` открытым (K21 UART жив, версии `k21AppInfo 3.6.24` читаются).
   Гипотезы: (а) asdkserver ждёт `ready=1` от `/dev/printer 0x7001` (~120с вызов,
   ~199с lockdown) — чинится put_user-дописыванием выше; (б) проверка лицензии
   (`lc licenseCode is null` в logcat!) — проверить; (в) K21-апдейт. Следующий шаг:
   собрать put_user-версию и вахтовать uptime 10 мин.
2. **Принтер**: цепочка `libCloudPos(/dev/printer ioctl 0xE0纸 power?) → libAsdkClient →
   asdkserver → UART ttyHSL1 → K21`. `/dev/ttymxc1` НЕ СУЩЕСТВУЕТ даже на стоке
   (мёртвая строка). Владелец `ioctl 0xE0` НЕ НАЙДЕН (не kingsee/misc_gpio/elmo/charger
   по дизассемблеру; единственный `cmp r0,#0xE0` во всём ядре — в PMIC-irq диспетче,
   мимо). **Осторожно: `ioctl(/dev/printer,0xE0,1)` дважды ронял кассу в EDL 9008**
   (просадка питания головки на слабом акке/кабеле?). Тесты питания только на
   заряженной кассе с хорошим кабелем! GPIO принтера неизвестен — узнать чтением:
   на стоке печатать чек и опрашивать `/sys/kernel/debug/gpio` (только чтение!).
   Приложение юзера для тестов: `github.com/Ssss4562/szzt-ks-print` (SDK-путь разобран
   в `docs/how-it-works.md`, растр 384 точки, GBK-шрифт).
3. **Экран при включении**: растянутая/полосатая заставка (`fifo 88881000/99991000`,
   `cmd 0x28 failed`), лечится выкл/вкл. DT панели/DSI/MDP/памяти — байт в байт
   со стоком (проверено диффом), дело в `mdp3_continuous_splash_on` adoption.
   Оба состояния cont-splash пробовали (без — чёрный экран, с — полосы). Глубокая
   работа по драйверу MDP3, на работу после загрузки не влияет.
4. **Камера**: `ov7251 NACK 0x70`, `ov2680 NACK 0x36`, `ov5648 chip mismatch`,
   `csid gdscr regulator`, `sysfs duplicate cam_vio` — не разбирали.
5. **WiFi**: стоковый `wlan.ko` (`version module_layout` mismatch) — пересобрать из
   исходников дерева под это ядро.

## Артефакты и источники правды
- `/home/user/kerneltests/` — все образы/логи (README.txt внутри).
- Стоковое ядро распаковано: `/tmp/opencode/stock_Image` (может быть затёрт —
  перераспаковать из `azurpos/boot.img`: gzip со смещения, `gzip -dc`).
- Бинарники демонов: `/tmp/opencode/{asdkserver,libAsdkClient.so,libCloudPos.so,
  libjni_szzt_deviceserver.so}` (стянуты с кассы) + `~/Downloads/szzt-ks8223-printer-hw.md`
  (разбор Клода: K21, UART, ioctl). Kallsyms стока + `su`/`iotest` (`/tmp/opencode/iotest.c`,
  static arm, класть в `/sbin` — `/data` noexec!) — рабочий метод RE.
- Ключевые строки стокового ядра: `k21_dev_ioctl ...`, `kingsee_gpio_plat_probe ok`,
  `hello elmo.`, `misc_gpio_ioctl`, DT `qcom,kingsee_gpio_misc` /
  `qcom,gpio_gen_3v3_power` / `qcom,gpio_k21_reset` / `qcom,FN_3V3_PWR`.
- База дизассемблера стокового Image: `file_offset = VA - 0xC0008000` (проверено по
  kallsyms + живым printk!). Не путать с 0xC0000000.
- Касса: Magisk ставится через TWRP (`twrp install /sdcard/Magisk-v25.2.zip`,
  zip уже на /sdcard). Прямой `adb reboot` глючит — перезапуск через рекавери.
  USB-ID: adb `05c6:9039`, fastboot `18d1:d00d`, EDL `05c6:900e`.
