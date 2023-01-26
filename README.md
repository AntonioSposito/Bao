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
PATH=/opt/arm-gnu-toolchain-12.2.rel1-x86_64-aarch64-none-elf/bin/aarch64-none-elf-:$PATH
export CROSS_COMPILE=/opt/arm-gnu-toolchain-12.2.rel1-x86_64-aarch64-none-elf/bin/aarch64-none-elf-
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
make -C $BAO_DEMOS_FREERTOS PLATFORM=$PLATFORM
```

Copio immagine finale di FreeRTOS nella cartella finale delle immagini dei guest

```
cp $BAO_DEMOS_FREERTOS/build/$PLATFORM/freertos.bin $BAO_DEMOS_WRKDIR_IMGS
```

### 6b. Build di Linux

#### Download source del kernel Linux

Setto variabili di ambiente per Linux

```
export BAO_DEMOS_LINUX=$BAO_DEMOS/guests/linux
export BAO_DEMOS_LINUX_REPO=https://source.codeaurora.org/external/imx/linux-imx
export BAO_DEMOS_LINUX_VERSION=rel_imx_5.4.24_2.1.0
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

