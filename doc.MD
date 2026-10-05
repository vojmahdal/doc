1. Připrav hostitelský systém

Doporučeno:

Shell
Ubuntu 24.04 LTS

Minimální HW:

Plain Text
CPU: 16 vláken+
RAM: 64 GB
Disk: 500 GB+

(Reálně kolem 250–350 GB pro BSP + Real-Time Edge build.)

2. Stáhni Real-Time Edge release

NXP používá vlastní manifest:

Shell
mkdir yocto-real-time-edge
cd yocto-real-time-edge
 
repo init \
-u https://github.com/nxp-real-time-edge-sw/yocto-real-time-edge.git \
-b real-time-edge-wrynose \
-m real-time-edge-3.5.0.xml
 
repo sync

Real-Time Edge obsahuje:

PREEMPT_RT kernel
AMP framework
Industrial networking stack
Jailhouse podporu
Yocto integraci
3. Inicializace buildu

FRDM board je součást konsolidovaného machine targetu.

Shell
export MACHINE=imx95evk
export DISTRO=fsl-imx-wayland
 
source real-time-edge-setup-env.sh -b build-imx95

V novějších release NXP vystupuje FRDM jako varianta:

Plain Text
imx95evk
├─ imx95-19x19-lpddr5-evk
├─ imx95-15x15-lpddr4x-evk
└─ imx95-15x15-lpddr4x-frdm

4. Přepni na FRDM board

V conf/local.conf:

Shell
MACHINE = "imx95-15x15-lpddr4x-frdm"

To je stejný machine target, který jste používali při Jailhouse evaluaci.

5. Zapni Jailhouse

Do local.conf přidejte:

Shell
DISTRO_FEATURES:append = " jailhouse"
 
IMXBOOT_VARIANT = "jailhouse"
 
IMAGE_INSTALL:append = " \
jailhouse-imx \
pyjailhouse \
"

Toto je konfigurace, kterou jste interně ověřili a generuje Jailhouse boot variantu.

6. Zapni L3 cache partitioning

AN14474 používá dodatečný layer:

Shell
https://github.com/nxp-imx-support/recipes-cachepartition

Ten obsahuje:

Plain Text
linux-imx patches
ATF patches
userspace utility

pro ovládání DSU cache partitioning.

Přidejte layer:

Shell
bitbake-layers add-layer ../recipes-cachepartition
7. Build Real-Time Edge image

Buildujte přímo Real-Time Edge image:

Shell
bitbake nxp-image-real-time-edge

nebo pokud release používá jiný název:

Shell
bitbake imx-image-real-time-edge
`

(ověřit přes bitbake -s).

Výsledný build bude obsahovat:

PREEMPT_RT
Real-Time Edge framework
Jailhouse balíčky
cache partitioning recipe
8. Ověř generované artefakty

V adresáři:

Shell
tmp/deploy/images/imx95-15x15-lpddr4x-frdm/

očekávej:

Plain Text
nxp-image-real-time-edge.wic
 
Image
 
imx-boot-variant-jailhouse-imx95-15x15-lpddr4x-frdm-sd.bin-flash_jailhouse
 
*.dtb

Jailhouse boot binárka má podle vašich testů název:

Plain Text
imx-boot-variant-jailhouse-imx95-15x15-lpddr4x-frdm-sd.bin-flash_jailhouse

9. Nahraj image na SD/eMMC

Pokud používáš SD:

Shell
sudo dd \
if=nxp-image-real-time-edge.wic \
of=/dev/sdX \
bs=8M status=progress
sync

Pro eMMC:

Shell
uuu -b emmc_all \
imx-boot-variant-jailhouse-imx95-15x15-lpddr4x-frdm-sd.bin-flash_jailhouse \
nxp-image-real-time-edge.wic

NXP workshop používal stejný princip flashování přes UUU.

10. Vytvoř chybějící FRDM root DTB

Tohle je aktuálně největší problém.

V interních poznámkách máte potvrzené, že:

pro FRDM není dodán Jailhouse root DTB a bylo nutné ho vytvořit z EVK varianty.

Postup:

Shell
scp root@board:/boot/imx95-15x15-evk-root.dtb .
scp root@board:/boot/imx95-15x15-evk.dtb .
scp root@board:/boot/imx95-15x15-frdm.dtb .

dekompilace:

Shell
dtc -I dtb -O dts \
-o imx95-15x15-frdm.dts \
imx95-15x15-frdm.dtb

a následné vytvoření

Plain Text
imx95-15x15-frdm-root.dts

podle rozdílů EVK vs FRDM.

11. Spusť Jailhouse

Na targetu:

Shell
modprobe jailhouse
Shell
export PATH=$PATH:/usr/share/jailhouse/tools/

Enable:

Shell
jailhouse enable \
/usr/share/jailhouse/cells/imx95.cell
`

Spuštění inmate Linuxu:

Shell
jailhouse cell linux \
/usr/share/jailhouse/cells/imx95-linux-demo.cell \
/boot/Image \
-d /boot/imx95-15x15-evk-inmate.dtb

Tento postup máte ověřený v interním Teams chatu.

12. Aktivuj L3 Cache Partitioning

Po bootu:

ověřit utility:

Shell
which usecache
`

nebo

Shell
ls /usr/bin | grep cache

AN14474 rozděluje L3 na:

Plain Text
4 way groups

Příklad:

Plain Text
CPU0-1 -> scheme 1
CPU2-5 -> scheme 0

tedy:

Plain Text
25 % cache
pro RT workload
 
75 % cache
pro Linux/Jailhouse root

NXP přímo uvádí tento scénář jako referenční konfiguraci.

13. Doporučená finální architektura

Pro demo MIMA bych šel do:

Root Cell
Plain Text
CPU0 CPU1 CPU2 CPU3
``
Linux
GPU
NPU server
Wayland
Inmate Cell
Plain Text
CPU4 CPU5
``
RT workload
PREEMPT_RT
dedikovaná L3 partition
Cache
Plain Text
Scheme 0
75 %
 
Scheme 1
25 %

přesně ve stylu AN14474.

Pozor na známý problém

Máte interně zdokumentovaný problém, že samotná Jailhouse boot binárka:

Plain Text
imx-boot-variant-jailhouse-imx95-15x15-lpddr4x-frdm-sd.bin-flash_jailhouse

ovlivňovala chování NPU i bez spuštění Jailhouse. NXP doporučovalo testovat přes jh_mmcboot. Pokud chceš kombinovat NPU + Jailhouse + cache partitioning, doporučuji nejdřív ověřit aktuální BSP (6.18.20+) a jestli NXP problém již opravilo.

Mohu také připravit konkrétní local.conf, bblayers.conf a patchset pro spojení Real-Time Edge + Jailhouse + AN14474 do jednoho reprodukovatelného buildu.
