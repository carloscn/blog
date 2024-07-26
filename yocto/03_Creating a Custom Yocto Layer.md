# [Yocto] Creating a Custom Yocto Layer

This wiki shows how to create a custom layer in an open source Xilinx Yocto flow.  After creating the layer, it provides an example of creating your own machine based on a ZCU102 configuration.

Please refer to original link : https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/57836605/Creating+a+Custom+Yocto+Layer

# Creating Your Custom Layer

You may create your custom layer manually by copying an existing `layer.conf`, however, Yocto provides some helper scripts to automate it.  The `bitbake-layers create-layer` script will generate a base layer with a default priority of 6.  Once the layer is created, you can either add the layer to `bblayers.conf` manually or use the `bitbake-layers add-layer` to automate it.  Note that adding it manually will be faster, but may increase the likelihood of typos or syntax errors.  The sequence below is an example of how to apply these scripts.  You may cut-and-paste the commands below into a local script to further automate this flow.  The resulting `layer.conf` and `bblayers.conf` are shown below.

## Setup the Bitbake environment.

```bash
source setupsdk
# Optional: Backup local.conf and bblayers.conf
cp conf/local.conf conf/local.conf.bk
cp conf/bblayers.conf conf/bblayers.conf.bk
```

## Create your custom layer.

```
LAYER=$ROOT/sources/meta-example
bitbake-layers create-layer $LAYER
mkdir -p $LAYER/{conf/machine,recipes-kernel/linux-xlnx,recipes-bsp/{u-boot/u-boot-xlnx,device-tree/files}}
bitbake-layers add-layer $LAYER
```

>The environment variable ROOT is defined in the "setupsdk" script.  ROOT is not defined in the Yocto proper "oe-init-build-env" script.

# Custom Layer Directory Structure

After the directory structure is created, you can begin populating the layer with **recipes, bbappends, patches, cfg fragments, machine configs,** etc.  The `example.bb` recipe is an artifact of the `create-layer` and can be safely deleted.  You can continue expanding your layer to include recipes for custom BSPs, applications and other open source components or stacks.  At this point, it's pretty straight forward to put your layer under Git control.

```console
$ tree meta-example
meta-example
├── conf
│   ├── layer.conf
│   └── machine
├── COPYING.MIT
├── README
├── recipes-bsp
│   ├── device-tree
│   │   └── files
│   └── u-boot
│       └── u-boot-xlnx
├── recipes-example
│   └── example
│       └── example.bb
└── recipes-kernel
    └── linux-xlnx
```

## Verify Your Layer

You can verify that your custom layer has been properly added to the Yocto build system by running `bitbake-layers` `show-layers` as shown below.

```console
$ bitbake-layers show-layers | grep meta-example
meta-example          /home/emil/yocto/sources/meta-example  6
```

## Removing Your Layer

To remove a layer, you can either manually edit the `bblayers.conf` or use the `bitbake-layers remove-layer` script as shown below from the `build` directory.

`$ bitbake-layers remove-layer meta-example`

## Creating a New Machine Configuration

Now lets create a machine configuration that inherits all of the defaults from the `zcu102-zynqmp` machine.  The name of this new machine will be the tuple `example-zcu102-zynqmp`.  In the `conf/machine` directory of your custom layer, create a machine configuration file named `example-zcu102-zynqmp.conf` with the `require` statement below .  This will include all of the defaults from the `zcu102-zynqmp`.  At this point you can customize the machine by adding new properties or overriding any of the `zcu102-zynqmp` defaults.

```
require conf/machine/zcu102-zynqmp.conf
```

Now you can bitbake your new machine.

`$ MACHINE=example-zcu102-zynqmp bitbake petalinux-image-minimal`
