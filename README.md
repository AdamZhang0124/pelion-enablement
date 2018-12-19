## Mbed OS Device Management / Pelion Enablement
How to bring up new platforms and run tests successfully (and with ease!)

### Setup your environment

1. Make sure that you have Mbed CLI over 1.8.3
   ```
   C:\>mbed --version
   1.8.3
   ```
   For instruction on how to upgrade Mbed CLI, please refer to this [link](https://github.com/ARMmbed/mbed-cli).

2. Get the whole test code via `mbed import git@github.com:ARMmbed/pelion-enablement.git`

   It will automatically expand dependent libraries in the test SW.

3. Get mbed_cloud_dev_credential.c file from the Pelion portal.

   Then overwrite the existing mbed_cloud_dev_credential.c file.

4. Install the `CLOUD_SDK_API_KEY`

   `mbed config -G CLOUD_SDK_API_KEY ak_1MDE1...<snip>`

   For convenience, this repository uses pre-generated certificates for API key `ak_1MDE1ZjZlMzg4ZTVkMDI0MjBhMDExYjA4MDAwMDAwMDA016617c30d482200d95670ee000000006iCt30Oe5HufoIQbyhTo1ybH00EZviYo`. It's recommended that you generate your own API key as some of the tests (e.g. LWM2M Post) are limited to 1 open active connection to the Pelion DM API server at a time. You should use this key only if you don't have access to Pelion Device Management (which is free to register for any Mbed developer - therefore go register now!).

   For instructions on how to generate your API key, please [see the documentation](https://cloud.mbed.com/docs/current/integrate-web-app/api-keys.html#generating-an-api-key).   

5. Initialize firmware credentials (optional)

   Simple Pelion DM Client provides Greentea tests to test your porting efforts. Before running these tests, it's recommended that you run the `mbed dm init` command, which will install all needed credentials for both Connect and Update Pelion DM features. You can use the following command:
   ```
   $ cd pelion-enablement
   $ mbed dm init -d "<your company name in Pelion DM>" --model-name "<product model identifier>" -q --force
   ```
   If above command do not work for your mbed-cli, please consider upgrading Mbed CLI.

6. (Optional) If you already compiled your bootloader for your specific target device

   If so, please update your bootloader to mbed-os.

### Running tests
NOTE: For Windows and GCC_ARM user, please refer to the section of 'Known problems'.

1. Compile tests for a target platform

```
mbed test -t <TOOLCHAIN> -m <TARGET> -n simple-mbed-cloud-client-tests-* --compile
```

2. Run tests for a platform

```
mbed test -t <TOOLCHAIN> -m <TARGET> -n simple-mbed-cloud-client-tests-* --run -v
```

#### Optional tests

The following tests may help you diagnoze issues if the tests above are failing 

1. Filesystem - to ensure that credentials and firmware can be stored.
```
mbed test -t <TOOLCHAIN> -m <TARGET> -n mbed-os-features-storage-tests-filesystem* -v
```

2. Blockdevice - to ensure that the layer under filesystem is running correctly
```
mbed test -t <TOOLCHAIN> -m <TARGET> -n mbed-os-features-storage-tests-blockdevice* -v
```

You can run both with:
```
mbed test -t ARM -m DETECT -n mbed-os-features-storage-tests* -v
```

### Adding a new target

#### 1. Create new target entry
Edit the `mbed_app.json` and create a new entry under `target_overrides`:
   * **Connectivity** - specify default connectivity type for your target. It's essential with targets that lack default connectivity set in targets.json or for targets that support multiple connectivity options. Example:
   ```
            "target.network-default-interface-type" : "ETHERNET",
   ```
   At the time of this writing, the possible options are `ETHERNET`, `WIFI`, `CELLULAR`.
   
   Depending on connectivity type, you might have to specify more config options, e.g. see already defined `CELLULAR` targets in `mbed_app.json`.

   * **Storage** - specify storage blockdevice type, which dynamically adds the blockdevice driver you specified at compile time. Example:
   ```
            "target.components_add" : ["SD"],
   ```
   Possible options are `SD`, `SPIF`, `QSPIF`, `FLASHIAP` (not recommended). Check more available options under https://github.com/ARMmbed/mbed-os/tree/master/components/storage/blockdevice

   You will also have to specify blockdevice pin configuration, which may be very different from one blockdevice type to another. Here's an example for `SD`:
   ```
            "sd.SPI_MOSI"                      : "PE_6",
            "sd.SPI_MISO"                      : "PE_5",
            "sd.SPI_CLK"                       : "PE_2",
            "sd.SPI_CS"                        : "PE_4",
   ```
   * **Flash** - define the basics for the microcontroller flash, e.g.:
   ```
            "flash-start-address"              : "0x08000000",
            "flash-size"                       : "(2048*1024)",
   ```
   * **SOTP** - define 2 SOTP/NVStore regions which will be used for Mbed OS Device Management to store it's special keys which are used to encrypt the data stored on the storage. Use the last 2 Flash sectors (if possible) to ensure that they don't get overwritten when new firmware is applied. Example:
   ```
            "sotp-section-1-address"            : "(MBED_CONF_APP_FLASH_START_ADDRESS + MBED_CONF_APP_FLASH_SIZE - 2*(128*1024))",
            "sotp-section-1-size"               : "(128*1024)",
            "sotp-section-2-address"            : "(MBED_CONF_APP_FLASH_START_ADDRESS + MBED_CONF_APP_FLASH_SIZE - 1*(128*1024))",
            "sotp-section-2-size"               : "(128*1024)",
            "sotp-num-sections" : 2
   ```
   `*-address` defines the start of the Flash sector and `*-size` defines the actual sector size. Currently `sotp-num-sections` should always be set to `2`.

   Note that these SOTP regions will be used for the next step...

#### 2. Recompile bootloader

1. Edit the `bootloader/bootloader_app.json` and specify:

   * **Flash** - define the basics for the microcontroller flash (the same as in `mbed_app.json`), e.g.:
    ```
            "flash-start-address"              : "0x08000000",
            "flash-size"                       : "(2048*1024)",
    ```

   * **SOTP** - similar to the **SOTP** step above, specify the location of the SOTP key storage. Note that in the bootloader, the variables are named differently. We should try to use the last 2 Flash sectors (if possible) to ensure that they don't get overwritten when new firmware is applied Example:
    ```
            "nvstore.area_1_address"           : "(MBED_CONF_APP_FLASH_START_ADDRESS + MBED_CONF_APP_FLASH_SIZE - 2*(128*1024))",
            "nvstore.area_1_size"              : "(128*1024)",
            "nvstore.area_2_address"           : "(MBED_CONF_APP_FLASH_START_ADDRESS + MBED_CONF_APP_FLASH_SIZE - 1*(128*1024))", "nvstore.area_2_size" : "(128*1024)",
    ```

    * **Application offset** - specify start address for the application and also the update-client meta info. As these are automatically calculated, you could copy the ones below:
    ```
            "update-client.application-details": "(MBED_CONF_APP_FLASH_START_ADDRESS + 64*1024)",
            "application-start-address"        : "(MBED_CONF_APP_FLASH_START_ADDRESS + 65*1024)",
            "max-application-size"             : "DEFAULT_MAX_APPLICATION_SIZE",
    ```
    
    * **Storage** - specify blockdevice pin configuration, exactly as you defined it in the `mbed_app.json` file. Example:
    ```
            "target.components_add"            : ["SD"],
            "sd.SPI_MOSI"                      : "PE_6",
            "sd.SPI_MISO"                      : "PE_5",
            "sd.SPI_CLK"                       : "PE_2",
            "sd.SPI_CS"                        : "PE_4"
    ```

2. Import the official [mbed-bootloader](https://github.com/ARMmbed/mbed-bootloader/) repository or the [mbed-bootloader-extended](https://github.com/ARMmbed/mbed-bootloader-extended/) repository that builds on top of `mbed-bootloader` and extends the support for filesystems and storage drivers. You can do this with ```mbed import mbed-bootloader-extended``` and change your commandline working dir, e.g. `cd mbed-bootloader-extended`.

3. Compile the bootloader using the `bootloader_app.json` configuration you just editted:
   ```
   mbed compile -t <TOOLCHAIN> -m <TARGET> --profile=tiny.json --app-config=<path to pelion-enablement/bootloader/bootloader_app.json>
   ```

   Note the following:
   * `mbed-bootloader` is primarily optimized for `GCC_ARM` and therefore you might want to compile it with that toolchain.
   * Before jumping to the next step, you should compile and flash the bootloader, and then connect over the virtual comport to ensure that the bootloader is running correctly. You can ignore errors related to checksum verification or falure to jump to application - these are expected at this stage.

#### 3. Include the bootloader
1. Copy the compiled bootloader from `mbed-bootloader/BUILDS/<TARGET>/<TOOLCHAIN>-TINY/mbed-bootloader.bin` to `pelion-enablement/bootloader/mbed-bootloader-<TARGET>.bin`.

2. Edit `pelion-enablement/mbed_app.json` and modify the target entry to include:
  ```
            "target.features_add"              : ["BOOTLOADER"],
            "target.bootloader_img"            : "bootloader/mbed-bootloader-<TARGET>.bin",
            "target.app_offset"                : "0x10400",
            "target.header_offset"             : "0x10000",
            "update-client.application-details": "(MBED_CONF_APP_FLASH_START_ADDRESS + 64*1024)",
   ```
 
   Note that:
   * `update-client.application-details` should be identical in both `bootloader_app.json` and `mbed_app.json`
   * `target.app_offset` is relative offset to `flash-start-address` you specified in the `mbed_app.json` and `bootloader_app.json`, and is the hex value of the offset specified by `application-start-address` in `bootloader_app.json`, e.g. `(MBED_CONF_APP_FLASH_START_ADDRESS+65*1024)` dec equals `0x10400` hex.
   * `target.header_offset` is also relative offset to the `flash-start-address` you specified in the `bootloader_app.json`, and is the hex value of the offset specified by `update-client.application-details`, e.g. `(MBED_CONF_APP_FLASH_START_ADDRESS+64*1024)` dec equals `0x10000` hex.

7. Finally, re-run all tests with:
```
mbed test -t <TOOLCHAIN> -m <TARGET> -n simple-mbed-cloud-client-tests-dev_mgmt*
```


#### Known problems

1. On Windows and when using `GCC_ARM`, you're likely to hit the Path limit. The workaround is running following commands:
    - Move the repo directory close to `C:\` 

      `move C:\...\pelion-enablement C:\P`
    - Rename simple-mbed-cloud-client foler and lib file
    
      `move simple-mbed-cloud-client spdmc`
      
      `move simple-mbed-cloud-client.lib spdmc.lib`
    - Compile and run with changed test name prefix of `spdmc-`.

      `mbed test -t <TOOLCHAIN> -m <TARGET> -n spdmc-tests-* --compile`
      
      `mbed test -t <TOOLCHAIN> -m <TARGET> -n spdmc-tests-* --run -v`
