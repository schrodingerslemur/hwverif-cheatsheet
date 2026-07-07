1. Create a directory ~/.local/bin
2. In directory, create a file ~/.local/bin/cadence.sh
3. In cadence.sh , paste this:
```
export XRUN_HOME=/afs/ece.cmu.edu/support/cds/share/image/usr/cds/xcelium-19.03.010/tools.lnx86/bin
export PATH="${XRUN_HOME}":$PATH
export VMANAGER_HOME=/afs/ece.cmu.edu/support/cds/share/image/usr/cds/vmanager-23.09/
export PATH="${VMANAGER_HOME}":$PATH
export PATH="${VMANAGER_HOME}/tools/bin":$PATH
export MDV_XLM_HOME=/afs/ece.cmu.edu/support/cds/share/image/usr/cds/xcelium-19.03.010/
export PATH="${MDV_XLM_HOME}":$PATH
export LM_LICENSE_FILE=/afs/ece/support/cds/share/image/usr/cds/share/license/license.dat
export CDS_LIC_FILE=5280@cadence-lic.ece.cmu.edu
export OA_BIT=64
export OA_UNSUPPORTED_PLAT=linux_rhel50_gcc44x
export CDS_AUTO_64BIT=ALL(edited)
```
4. Edit your ~/.bashrc , and add this to the last line source ~/.local/bin/cadence.sh
5. Restart terminal, and try to run xrun it should show something
6. To run with cadence, do `xrun <file_name>.sv <file_name>.sv`, you don't have to compile and do `./simv` anymore. Just running xrun does both steps for you
7. For waveform, do `xrun -debug -gui <file_name>.sv`
