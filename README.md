## BAO

Home del progetto, basato su [Bao-project](https://github.com/bao-project)

[Nuovo readme](https://github.com/AntonioSposito/Bao/blob/main/automake.md) con la guida per buildare bao con un guest baremetal

[Vecchio readme](https://github.com/AntonioSposito/Bao/blob/main/README_old.md) con la guida su come buildare bao con freertos+linux

### File presenti nella repo:

-`compilebaremetal.sh` script per compilare solamente il guest baremetal

-`compilebao.sh` script per compilare bao da lanciare ogni volta che si vuole ricompilare bao oppure dopo aver ricompilato il guest

-`launchqemu.sh` lancia bao tramite qemu, setta anche la connessione per il debug con gdb
