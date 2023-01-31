# Bao hypervisor con guest baremetal

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
export DEMO=baremetal
export PLATFORM=qemu-aarch64-virt
export ARCH=aarch64
```

## 5. Esecuzione auto make

Eseguire:
```
make -j$(nproc)
```
Che scaricherà i file sorgente per la piattaforma e la demo indicata, compilerà il tutto e produrrà il file finale bao.bin che sarà quello passato a qemu

## 6. Qemu

Per lanciare qemu basta eseguire:
```
make run
```
A schermo verranno stampato il log di u-boot, per far partire bao dobbiamo dire a u-boot di saltare alla locazione di memoria dove lo abbiamo salvato:

```
go 0x50000000
```
