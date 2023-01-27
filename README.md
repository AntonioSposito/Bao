# Bao hypervisor con linux + freertos

## 1. Scaricare pacchetti 

Scarico pacchetti necessari alla compilazione di bao, linux e freertos

```
sudo apt install build-essential bison flex git libssl-dev ninja-build u-boot-tools pandoc libpixman-1-dev libglib2.0-dev
```

## 2. Download cross-compilatore

Scaricare cross compilatore da [questo sito](https://developer.arm.com/downloads/-/arm-gnu-toolchain-downloads), io ho scaricato quello per host Linux x86_64 e bare metal aarch 64 target machine.
Una volta scompattato l'archivio posiziono il conenuto nella cartella /opt del mio pc

## 3. Clono repository github di bao 

Questi comandi clonano la repository di bao e la posizonano in una cartelal denominata bao-demos

```
git clone https://github.com/bao-project/bao-demos
cd bao-demos
```

## 4. Setto le variabili di ambiente

Per la compilazione di bao ho necessità di andare a settare alcune variabili di ambiente, andiamo a settare tutto per avere come piattaforma target Qemu, sul quale faremo girare una vm contenente Linux e un'altra vm contenente freertos.

- CROSS_COMPILE deve essere settata in base alla versione del compilatore scaricato
- Le altre variabili vanno settate in base all'[appendice](https://github.com/bao-project/bao-demos#Appendix-I) della guida ufficiale di bao.

```
PATH=/opt/gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf/bin-:$PATH
export CROSS_COMPILE=/opt/gcc-arm-10.3-2021.07-x86_64-aarch64-none-elf/bin/aarch64-none-elf-
export DEMO=linux+freertos
export PLATFORM=qemu-aarch64-virt
export ARCH=aarch64
```

## 5. Creo cartelle di lavoro

Creo cartelle di lavoro dove verrà piazzato il codice e le immagini del progetto:

```
export BAO_DEMOS=$(realpath .)
export BAO_DEMOS_WRKDIR=$BAO_DEMOS/wrkdir
export BAO_DEMOS_WRKDIR_SRC=$BAO_DEMOS_WRKDIR/srcs
export BAO_DEMOS_WRKDIR_PLAT=$BAO_DEMOS_WRKDIR/imgs/$PLATFORM
export BAO_DEMOS_WRKDIR_IMGS=$BAO_DEMOS_WRKDIR_PLAT/$DEMO
mkdir -p $BAO_DEMOS_WRKDIR
mkdir -p $BAO_DEMOS_WRKDIR_SRC
mkdir -p $BAO_DEMOS_WRKDIR_IMGS
```

## 6. Build dei guest 

### 6a. Build di Freertos

Setto una variaible di ambiente che mi servirà per freertos

```
export BAO_DEMOS_FREERTOS=$BAO_DEMOS_WRKDIR_SRC/freertos
```

Clono repository e buildo

```
git clone --recursive --shallow-submodules\
    https://github.com/bao-project/freertos-over-bao.git\
    $BAO_DEMOS_FREERTOS --branch demo
```
Prima di lanciare il make, bisogna modificare un file che contiene un nome di un registro errato, come indicato [qui](https://github.com/bao-project/bao-baremetal-guest/pull/1/commits/033220718933c328ac508ac084f1f89bb4523024). Il file da cambiare si trova in: bao-demos/wrkdir/srcs/freertos/src/baremetal-runtime/src/arch/armv8/aarch64/ e si chiama start.S, in particolare la riga 57 va modificata in: 

```
msr ICC_SRE_EL2, x1
```
Una volta modificato il file start.S possiamo compilare freertos

```
make -C $BAO_DEMOS_FREERTOS PLATFORM=$PLATFORM
```
Copio immagine finale di FreeRTOS (freertos.bin) nella cartella delle immagini dei guest (bao-demos/wrkdir/imgs/qemu-aarch64-virt/linux+freertos)

```
cp $BAO_DEMOS_FREERTOS/build/$PLATFORM/freertos.bin $BAO_DEMOS_WRKDIR_IMGS
```

### 6b. Build di Linux

#### Download source del kernel Linux

Setto variabili di ambiente per Linux

```
export BAO_DEMOS_LINUX=$BAO_DEMOS/guests/linux
export BAO_DEMOS_LINUX_REPO=https://github.com/torvalds/linux.git
export BAO_DEMOS_LINUX_VERSION=v5.11
export BAO_DEMOS_LINUX_SRC=$BAO_DEMOS_WRKDIR_SRC/linux-$BAO_DEMOS_LINUX_VERSION
```

Clono repository 

```
git clone $BAO_DEMOS_LINUX_REPO $BAO_DEMOS_LINUX_SRC\
    --depth 1 --branch $BAO_DEMOS_LINUX_VERSION
cd $BAO_DEMOS_LINUX_SRC
git apply $BAO_DEMOS_LINUX/patches/$BAO_DEMOS_LINUX_VERSION/*.patch
```

Setto variabili di ambiente che puntano all'architettura target e alla specifica piattaforma scelte da noi

```
export BAO_DEMOS_LINUX_CFG_FRAG=$(ls $BAO_DEMOS_LINUX/configs/base.config\
    $BAO_DEMOS_LINUX/configs/$ARCH.config\
    $BAO_DEMOS_LINUX/configs/$PLATFORM.config 2> /dev/null)
```

#### Build di linux con initramfs incorporato con buildroot

Setto variabili di ambiente:

```
export BAO_DEMOS_BUILDROOT=$BAO_DEMOS_WRKDIR_SRC/\
buildroot-$ARCH-$BAO_DEMOS_LINUX_VERSION
export BAO_DEMOS_BUILDROOT_DEFCFG=$BAO_DEMOS_LINUX/buildroot/$ARCH.config
export LINUX_OVERRIDE_SRCDIR=$BAO_DEMOS_LINUX_SRC
```

Clono l'ultima versione stabile di buildroot


```
git clone https://github.com/buildroot/buildroot.git $BAO_DEMOS_BUILDROOT\
    --depth 1 --branch 2020.11.3
cd $BAO_DEMOS_BUILDROOT
```

Uso il file defconfig scaricato clonando le repository per patchare, buildo il tutto


```
make defconfig BR2_DEFCONFIG=$BAO_DEMOS_BUILDROOT_DEFCFG
make linux-reconfigure all
mv $BAO_DEMOS_BUILDROOT/output/images/Image\
    $BAO_DEMOS_BUILDROOT/output/images/Image-$PLATFORM
```
#### Build del device tree

```
export BAO_DEMO_LINUX_VM=linux
dtc $BAO_DEMOS/demos/$DEMO/devicetrees/$PLATFORM/$BAO_DEMO_LINUX_VM.dts >\
    $BAO_DEMOS_WRKDIR_IMGS/$BAO_DEMO_LINUX_VM.dtb
make -C $BAO_DEMOS_LINUX/lloader\
    ARCH=$ARCH\
    IMAGE=$BAO_DEMOS_BUILDROOT/output/images/Image-$PLATFORM\
    DTB=$BAO_DEMOS_WRKDIR_IMGS/$BAO_DEMO_LINUX_VM.dtb\
    TARGET=$BAO_DEMOS_WRKDIR_IMGS/$BAO_DEMO_LINUX_VM.bin
```

A questo punto i file linux.bin, linux.elf e linux.dtb sarà posizionati nella cartella /bao-demos/wrkdir/imgs/qemu-aarch64-virt/linux+freertos

## 7. Build di bao

Clono repository di bao
```
export BAO_DEMOS_BAO=$BAO_DEMOS_WRKDIR_SRC/bao
git clone https://github.com/bao-project/bao-hypervisor $BAO_DEMOS_BAO\
    --branch v0.1.0
```
Copio configurazione di sistema nelle working directory

```
mkdir -p $BAO_DEMOS_WRKDIR_IMGS/config
cp -L $BAO_DEMOS/demos/$DEMO/configs/$PLATFORM.c\
    $BAO_DEMOS_WRKDIR_IMGS/config/$DEMO.c
```
Build di bao:
```
make -C $BAO_DEMOS_BAO\
    PLATFORM=$PLATFORM\
    CONFIG_REPO=$BAO_DEMOS_WRKDIR_IMGS/config\
    CONFIG=$DEMO\
    CONFIG_BUILTIN=y\
    CPPFLAGS=-DBAO_DEMOS_WRKDIR_IMGS=$BAO_DEMOS_WRKDIR_IMGS
```
Sposto l'immagine bao.bin nella cartella /bao-demos/wrkdir/imgs/qemu-aarch64-virt/linux+freertos
```
cp $BAO_DEMOS_BAO/bin/$PLATFORM/builtin-configs/$DEMO/bao.bin\
    $BAO_DEMOS_WRKDIR_IMGS
```

## 8. Qemu

### 8a. Build

Download e installazione di qemu:
```
export BAO_DEMOS_QEMU=$BAO_DEMOS_WRKDIR_SRC/qemu-$ARCH
git clone https://github.com/qemu/qemu.git $BAO_DEMOS_QEMU --depth 1\
   --branch v6.1.0
cd $BAO_DEMOS_QEMU
./configure --target-list=aarch64-softmmu
make -j$(nproc)
sudo make install
```
Preparo il file .config
```
export BAO_DEMOS_UBOOT=$BAO_DEMOS_WRKDIR_SRC/u-boot
git clone https://github.com/u-boot/u-boot.git $BAO_DEMOS_UBOOT --depth 1\
   --branch v2021.01
cd $BAO_DEMOS_UBOOT
make qemu_arm64_defconfig
```
Modifico file file .config con queste due linee di codice:
```
echo "CONFIG_TFABOOT=y" >> .config
echo "CONFIG_SYS_TEXT_BASE=0x60000000" >> .config
```
Build di uboot
```
make -j$(nproc)
```
Copia di u-boot.bin in /bao-demos/wrkdir/imgs/qemu-aarch64-virt/
```
cp $BAO_DEMOS_UBOOT/u-boot.bin $BAO_DEMOS_WRKDIR/imgs/$PLATFORM
```
Build TF-A
```
export BAO_DEMOS_ATF=$BAO_DEMOS_WRKDIR_SRC/arm-trusted-firmware
git clone https://github.com/bao-project/arm-trusted-firmware.git\
   $BAO_DEMOS_ATF --depth 1
cd $BAO_DEMOS_ATF
make PLAT=qemu bl1 fip BL33=$BAO_DEMOS_WRKDIR/imgs/$PLATFORM/u-boot.bin\
   QEMU_USE_GIC_DRIVER=QEMU_GICV3
dd if=$BAO_DEMOS_ATF/build/qemu/release/bl1.bin\
   of=$BAO_DEMOS_WRKDIR/imgs/$PLATFORM/flash.bin
dd if=$BAO_DEMOS_ATF/build/qemu/release/fip.bin\
   of=$BAO_DEMOS_WRKDIR/imgs/$PLATFORM/flash.bin seek=64 bs=4096 conv=notrunc
```

### 8b. Avvio di qemu

```
qemu-system-aarch64 -nographic\
   -M virt,secure=on,virtualization=on,gic-version=3 \
   -cpu cortex-a53 -smp 4 -m 4G\
   -bios $BAO_DEMOS_WRKDIR/imgs/$PLATFORM/flash.bin \
   -device loader,file="$BAO_DEMOS_WRKDIR_IMGS/bao.bin",addr=0x50000000,force-raw=on\
   -device virtio-net-device,netdev=net0 -netdev user,id=net0,hostfwd=tcp:127.0.0.1:5555-:22\
   -device virtio-serial-device -chardev pty,id=serial3 -device virtconsole,chardev=serial3
```
Oppure posizionarsi in /bao-demos e lanciare:
```
make run
```

Una volta lanciata l'emulazione, viene visualizzato a schermo  il nome del pseudo-terminale in cui sarà piazzata la virtio seriale, con una riga del genere `char device redirected to /dev/pts/4 (label serial3)`. In una shell qualsiasi di linux, una volta avviato qemu si può lanciare il seguente comando per connettersi al pseudo-terminale:

```
screen /dev/pts/4
```

Oppure, ci si può collegare via ssh con il seguente comando:
```
ssh root@127.0.0.1 -p 5555
```
E usare username `root` e password `root` per accedere a linux.











