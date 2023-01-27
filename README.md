# Bao hypervisor con linux + freertos

## 1. Scaricare pacchetti 

Scarico pacchetti necessari alla compilazione di bao, linux e freertos

```
sudo apt install build-essential bison flex git libssl-dev ninja-build u-boot-tools pandoc libpixman-1-dev
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

## 8. Build di qemu


















