
![](https://raw.githubusercontent.com/carloscn/images/main/typora202407291548530.png)

```bash
do_deploy_append() {
	install -d ${DEPLOY_DIR_IMAGE}
	# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
	# >>>>>>>>>>>>>>>>>> Note, this block in this script is for secure_boot <<<<<<<<<<<<<<<<<<<
	# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
	# Get uboot entry point address from
	pushd ${B}/${config} > /dev/null
	./tools/mkimage -n board/freescale/mx6sabresd/icm7101-ddr-solo.cfg.cfgtmp -T imximage -e 0x17800000 -d u-boot.bin u-boot.imx > u-boot.imx.log
	popd > /dev/null
	# Copy u-boot.imx.log to deploy/images directory
	install -m 0777 ${B}/${config}/u-boot.imx.log ${DEPLOY_DIR_IMAGE}/u-boot.imx.log
	# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
	# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> Secure Boot End <<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<
	# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

	fwpack -o ${DEPLOY_DIR_IMAGE}/${PRODUCT_NAME}_uboot_${BOARD_NAME}.bin -P "${PRODUCT_NAME}" \
		-M "${MODEL_NAME}" -B "${BOARD_NAME}" \
		-S "${FIRMWARE_VERSION}" -D "${RELEASE_DATE}" \
		-U "${UBOOT_VERSION}" -u ${UBOOT_BINARY}
}
```


![](https://raw.githubusercontent.com/carloscn/images/main/typora202407301416908.png)
