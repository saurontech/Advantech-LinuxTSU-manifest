# This is the manifest of the Advantech Linux TSU Yocto project
The goal of this project is to release opensource Linux source code that runs on Advantech Hardware

# Download the Yocto Linux for ECU-150-A1 and setup environment variables
For downloading the source code and setting up the environment, follow the instructions below:
```sh
foo@bar:~/yocto$ repo init -u https://github.com/saurontech/Advantech-LinuxTSU-manifest.git -b main -m ecu-150a1-6.6.36-2.1.0.xml
foo@bar:~/yocto$ repo sync
foo@bar:~/yocto$ MACHINE=imx8mq-ecu150a1 DISTRO=fsl-imx-xwayland source ./imx-setup-release.sh -b build
foo@bar:~/yocto/build$
```
After the commands, one has not only downloaded the Yocto project for ECU-150-A1, the environment variables, of the operating console, were also setup to operate bitbaker commands.
Please also notice, that after the commands, your current position has been changed to the build directory!
If, in the future, one wishes to setup environment variables of another console; in that spacific console, one should use the command:
```sh
foo@bar:~/yocto$ source ./setup-environment build
foo@bar:~/yocto/build$
```
# Building Yocto
In a console that has been setup properly, use the following command to build Linxu and the Yocto rootfs
```sh
foo@bar:~/yocto/build/$ bitbaker core-image-minimal

```
The Linux kernel will be located at : ./build/tmp/deploy/images/imx8mq-ecu150a1/Image
The dtb will be located at: ./build/tmp/deploy/images/imx8mq-ecu150a1/fsl-imx8mq-ecu150a1.dtb
The rootfs will be located at: ./build/tmp/deploy/images/imx8mq-ecu150a1/core-image-minimal-imx8mq-ecu150a1.rootfs.tar.gz

If one only wishes to build the linux kernel, in the build directory, use the following command:
```sh
foo@bar:~/yocto/build$ bitbaker linux-imx
```

