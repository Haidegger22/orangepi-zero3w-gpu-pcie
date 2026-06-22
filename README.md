# Orange Pi Zero 3W — GPU + PCIe + NPU на Debian 13

> **GPU РАБОТАЕТ на Debian 13 (Trixie)!** 🎉
>
> Сборка `pvrsrvkm.ko` + отложенная загрузка через systemd-сервис (udev вызывает kernel panic на раннем этапе).
> Проверено 22 июня 2026.

## Статус компонентов

| Компонент | Статус | Версия / Детали |
|-----------|:------:|--------|
| Ядро | ✅ | 6.6.98-sun60iw2 |
| WiFi (AIC8800D80) | ✅ | из коробки |
| Bluetooth | ✅ | из коробки |
| HDMI (видео+звук) | ✅ | 1360×768, из коробки |
| USB 3.0 + NVMe (JMS583) | ✅ | 660 MB/s |
| **GPU pvrsrvkm** | ✅ | собран, загружается через systemd-сервис |
| **/dev/dri** | ✅ | card0 (HDMI) + card1 (GPU) + renderD128 |
| **OpenGL ES** | ❌ | нет userspace (нужен pvr_dri.so из Radxa) |
| **Vulkan** | ❌ | нет userspace (нужен libVK_IMG.so из Radxa) |
| **NPU** | ✅ | /dev/vipcore, vipcore модуль загружен |
| **PCIe 3.0 x1** | ✅ | контроллер активен, нужен FPC-адаптер |

## 🔬 Ключевые открытия (22 июня 2026)

### 1. DTB Bullseye и Trixie — идентичны
Декомпилировали DTB из Debian 11 (Bullseye) и Debian 13 (Trixie):
- 10332 строки в обоих
- `diff` — пустой
- `power-domains` для GPU (domain #5 и #6) есть в обоих DTB

**Вывод:** Проблема НЕ в device tree. Ранний диагноз (20 июня) был ошибочным.

### 2. Стоковый pvrsrvkm.ko — битый
В Debian 13 лежит `/lib/modules/.../kernel/drivers/gpu/drm/img-rogue/pvrsrvkm.ko` (23.8 MB).
- `modinfo`: Invalid argument
- `insmod`: Invalid module format

**Orange Pi намеренно положили битый модуль как предохранитель** — чтобы udev не загрузил рабочий модуль слишком рано и не вызвал kernel panic. На Debian 11 модуль рабочий, на Debian 13 — битый.

### 3. Причина kernel panic — не DTB, а порядок загрузки
- `insmod` в ЖИВОЙ системе (после полной загрузки): ✅ работает
- Автозагрузка через udev на старте ядра: ❌ kernel panic

Модуль пытается обратиться к GPU до инициализации питания/клоков → система падает.

### 4. Решение: отложенная загрузка через systemd
- Модуль кладётся в `/opt/pvrsrvkm.ko` (udev не видит)
- systemd-сервис с `After=multi-user.target` загружает модуль после рабочего стола
- Работает стабильно через перезагрузки

---

## 📦 Пошаговая инструкция: сборка и установка GPU

> Все команды выполняются прямо на Orange Pi Zero 3W (Debian 13, ядро 6.6.98-sun60iw2).

### Шаг 1: Установить заголовки ядра

```bash
# Заголовки уже лежат в /opt после установки системы
echo orangepi | sudo -S dpkg -i /opt/linux-headers-current-sun60iw2_1.0.0_arm64.deb
```

### Шаг 2: Скачать исходники GPU-модуля

```bash
cd /tmp && rm -rf linux-gpu
git clone --depth 1 --branch orange-pi-6.6-sun60iw2 --filter=blob:none --sparse \
  https://github.com/orangepi-xunlong/linux-orangepi.git linux-gpu
cd /tmp/linux-gpu
git sparse-checkout set bsp/modules/gpu
```

### Шаг 3: Создать заглушку sunxi-sid.h

> Заголовок `sunxi-sid.h` отсутствует в установленных хедерах ядра.
> Он нужен для DVFS-efuse. Возвращаем 0 (использовать стандартные напряжения).

```bash
echo orangepi | sudo -S mkdir -p /usr/src/linux-headers-6.6.98-sun60iw2/include/soc/sunxi
echo orangepi | sudo -S tee /usr/src/linux-headers-6.6.98-sun60iw2/include/soc/sunxi/sunxi-sid.h << 'EOF'
#ifndef __SUNXI_SID_H
#define __SUNXI_SID_H
#include <linux/types.h>
#include <linux/errno.h>
#define SUNXI_CHIP_SUN60IW2   (0x17330000)
#define SUN60IW2P1_REV_A SUNXI_CHIP_REV(SUNXI_CHIP_SUN60IW2, 0x0)
#define SUNXI_CHIP_REV(p, v)  (p + v)
static inline int sunxi_get_module_param_from_sid(void *buf, int offset, int len) { return 0; }
static inline int sunxi_get_soc_chipid(u32 *chipid) { if (chipid) *chipid = 0; return 0; }
#endif
EOF

# Скопировать в include/linux для angle-bracket includes
echo orangepi | sudo -S cp /usr/src/linux-headers-6.6.98-sun60iw2/include/soc/sunxi/sunxi-sid.h \
  /usr/src/linux-headers-6.6.98-sun60iw2/include/linux/sunxi-sid.h
```

### Шаг 4: Собрать модуль

```bash
cd /tmp/linux-gpu/bsp/modules/gpu/img-bxm/linux/rogue_km

# Фикс совместимости с make 4.4+
sed -i 's/^\.SECONDARY.*//g' \
  build/linux/kbuild/Makefile.template \
  build/linux/toplevel.mk \
  build/linux/defs.mk

# Фаза 1: Imagination build system
export LICHEE_TOOLCHAIN_PATH=/usr/bin/
make -C build/linux/sunxi_linux ARCH=arm64 \
  KERNELDIR=/lib/modules/$(uname -r)/build \
  BUILD=release

# Фаза 1 упадёт на sunxi_platform.c — это нормально.
# Фикс: заглушка sunxi-sid.h + quoted include
KBUILD=binary_sunxi_linux_nulldrmws_release/target_aarch64/kbuild
cp /usr/src/linux-headers-6.6.98-sun60iw2/include/soc/sunxi/sunxi-sid.h \
   $KBUILD/services/system/rogue/rgx_sunxi/sunxi-sid.h
sed -i 's|#include <sunxi-sid.h>|#include "sunxi-sid.h"|' \
   $KBUILD/services/system/rogue/rgx_sunxi/sunxi_platform.c

# Ребилд (быстрый — .o файлы уже скомпилированы)
make -C build/linux/sunxi_linux ARCH=arm64 \
  KERNELDIR=/lib/modules/$(uname -r)/build \
  BUILD=release

# Проверить
modinfo $KBUILD/pvrsrvkm.ko | grep vermagic
# Должен показать: vermagic=6.6.98-sun60iw2 SMP preempt mod_unload aarch64
```

### Шаг 5: Установить модуль (отложенная загрузка)

```bash
# 1. Копируем в /opt (НЕ в /lib/modules — udev не должен видеть)
echo orangepi | sudo -S cp $KBUILD/pvrsrvkm.ko /opt/pvrsrvkm.ko

# 2. Создаём systemd-сервис
echo orangepi | sudo -S tee /etc/systemd/system/gpu-late.service << 'EOF'
[Unit]
Description=Load GPU driver after system is fully up
After=multi-user.target
Wants=multi-user.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/sbin/insmod /opt/pvrsrvkm.ko
ExecStop=/sbin/rmmod pvrsrvkm
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# 3. Включить сервис
echo orangepi | sudo -S systemctl daemon-reload
echo orangepi | sudo -S systemctl enable gpu-late.service

# 4. Перезагрузить
echo orangepi | sudo -S reboot
```

### Шаг 6: Проверка после перезагрузки

```bash
lsmod | grep pvr          # pvrsrvkm должен быть загружен
ls /dev/dri/               # card0 (HDMI) + card1 (GPU) + renderD128
systemctl status gpu-late  # active (exited), SUCCESS
```

---

## 📁 Файлы

- **pvrsrvkm.ko** → `/opt/pvrsrvkm.ko` (udev не трогает)
- **systemd-сервис** → `/etc/systemd/system/gpu-late.service`
- **Стоковый битый модуль** → `/lib/modules/.../kernel/drivers/gpu/drm/img-rogue/pvrsrvkm.ko` (не трогаем)
- **Заглушка** → `/usr/src/linux-headers-6.6.98-sun60iw2/include/soc/sunxi/sunxi-sid.h`

---

## 🔮 Что дальше (userspace GPU)

Нужно установить userspace для OpenGL ES и Vulkan:
1. **Firmware GPU**: `rgx.fw.*` + `rgx.sh.*` → `/lib/firmware/`
2. **Mesa DRI**: `pvr_dri.so` → `/usr/local/lib/dri/`
3. **Vulkan ICD**: `libVK_IMG.so` + зависимости → `/usr/lib/`
4. **PVR библиотеки**: `libpvr*`, `libPVR*`, `libGLESv2_PVR*` → `/usr/lib/`
5. **Vulkan ICD JSON**: `/usr/share/vulkan/icd.d/pvr_icd.json`

Источник: [Radxa allwinner-target](https://github.com/radxa/allwinner-target) (ветка `target-a733-v1.4.6`, каталог `debian/cubie_a7z/overlay/`)

---

## 🗓 Хронология

| Дата | Событие |
|------|---------|
| 15-16 июня | Ложный успех: модуль собрался, insmod работал, но без перезагрузки |
| 20 июня | Kernel panic при перезагрузке. Ошибочный диагноз: «power-domains в DTS» |
| 21 июня | Изучение Incipiens/OrangePiZero3W-GPU-VPU. Работа над ошибками. |
| 22 июня | **Прорыв:** DTB Bullseye=Trixie (diff пустой). Стоковый модуль битый (предохранитель). Сборка с заглушкой sunxi-sid.h. Отложенная загрузка через systemd → GPU работает без паники |
| 22 июня | README обновлён |

## Источники

- Ядро: https://github.com/orangepi-xunlong/linux-orangepi (ветка `orange-pi-6.6-sun60iw2`)
- Референс: https://github.com/Incipiens/OrangePiZero3W-GPU-VPU
- GPU userspace (будущее): https://github.com/radxa/allwinner-target (ветка `target-a733-v1.4.6`)
- Наш репозиторий: https://github.com/Haidegger22/orangepi-zero3w-gpu-pcie
