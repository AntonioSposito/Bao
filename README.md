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

