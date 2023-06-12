# Run Linux and PostMarket OS on the iPhone 5s

## Prerequisites

The version of usbmuxd that ships with most distros is broken. Use [usbmuxd2](https://github.com/tihmstar/usbmuxd2)

PostmarketOS' development tool [pmbootstrap](https://wiki.postmarketos.org/wiki/Pmbootstrap) is installed

## Jailbreak and load PongoOS

***NOTE:*** **Read all the following steps before proceeding, as some quicktime events will occur during your quest, and it's better you're ready for them**

1. Connect the iPhone in recovery mode by holding down the power and home buttons until the phone reboots and you see the iTunes logo appear
2. Run the checkra1n exploit from the base directory of this repo with

    ```sh
    sudo ./scripts/checkra1n -v -V -p -k ./resources/Pongo.bin
    ```

    and follow the on-screen instructions

    NOTE: There are multiple options for the PongoOS image:
    - [Pongo.bin](./resources/Pongo.bin) - The default PongoOS image, capable of booting Linux. SMC calls cause hangs.
    - [Pongo-Empty-EL3.bin)](./resources/Pongo-Empty-EL3.bin) - PongoOS image with the ARM Exception Vector empty(`eret` in all 16 entries). SMC calls don't crash it, and don't do anything. Comes with the `el3_smc_call` command to test the non-crashing behaviour in `pongoterm`.
    - [Pongo-EL3-fibonacci.bin](./resources/Pongo-EL3-fibonacci.bin) PongoOS with a basic EL3 runtime that computes a fibonacci number. Register saving and restoring on EL3 entry/exit work. Comes with the `el3_fib [number]` command to test this in `pongoterm`. The Fibonacci EL3 service can be used in Linux through the [el3_demo_fib](https://gitlab.inf.ethz.ch/PRV-SHINDE/theses/ma-theses/ma-2022/221114_lyubomir_kyorovski/iphone-5s/lkm-el3-fib-svc) kernel module (See [this section](#el3-fibonacci-service-lkm))
3. When you get to "Right before trigger":
   1. Wait 3 seconds (or count to 3)
   2. Unplug iPhone
   3. Wait 3 seconds
   4. Re-plug iPhone
   5. The message should change to "Booting..."
        - if it doesn't - keep re-plugging until it does
        - if booting to PongoOS fails outright and the iPhone boots normally, restart the whole jailbreak procedure
4. With any luck, you should be now in PongoOS
   - You can poke around it with `pongoterm` which you can compile from [here](https://gitlab.inf.ethz.ch/PRV-SHINDE/theses/ma-theses/ma-2022/221114_lyubomir_kyorovski/iphone-5s/pongoos-5s/-/tree/master/scripts)

## Booting the Linux kernel and into an initramfs

After jailbreaking, we are left with a pre-boot environment. We can utilise it to load something more complete

1. Still connected to the iPhone and in PongoOS, run the following command from the root of this repo

    ```sh
    sudo python3 ./scripts/load_linux.py -k ./resources/Image.lzma -d ./resources/dtbpack -r ./resources/nbramdisk.img
    ```

2. After a few seconds, you will see the kernel boot log and land on a PostmarketOS loading screen
3. The `initramfs` is loaded now and you should see a warning about a ***debug-shell*** being enabled. You can now connect to the initramfs using telnet:

   ```sh
   telnet 172.16.42.1
   ```

    It comes with busybox pre-installed, so you can be fairly creative with what to try next.

## Boot PostmarketOS (with GUI!)

The ramdisk we loaded allows us to netboot a proper linux distro image. A good choice for that is [PostmarketOS](https://postmarketos.org/)

1. To get a compatible image, for now we can use the `pmbootstrap` utility:

    ```sh
    pmbootstrap init
    ```

2. When prompted for configuration options, select the following(you can take the suggested options for the ones not listed here):

    ```cfg
    Channel [edge]
    Vendor [apple]
    Device codename [iphone6]
    Username [ssp]
    User interface [xfce4]
    Device hostname [ssp-iphone5s]
    ```

3. To build the image, run

    ```sh
    pmbootstrap install
    ```

    - When prompted for a password, use something easy to remember (like ***"ssp"***)
4. Connect to the ramdisk and make it start the netboot listener

    ```sh
    telnet 172.16.42.1
    > pmos_continue_boot
    ```

5. You should see a "waiting for netboot" message on the iphone now. We can use `pmbootstrap` to serve the image. **Run the following command in a new terminal and keep it open**:

   ```sh
   pmbootstrap netboot serve
   ```

   - **NOTE:** If nothing happens after this, likely your firewall blocks communication to the iPhone. To resolve this, find the network interface of the iPhone with

     ```sh
     ip -br -4 a sh | grep 172.16.42.2 | awk '{print $1}'
     ```

     and remove any restrictions on it, e.g. under Fedora Linux, you could use

     ```sh
     firewall-cmd --zone=trusted --change-interface=$INTERFACE_NAME
     ```

6. You should get a tty, at which point you can ***connect with SSH***, using the password you specified earlier

    ```sh
    ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password ssp@172.16.42.1
    ```

7. The last step is to start the GUI. Take note of the tty displayed on the iPhone's screen (should be `tty1`). Connect to this tty using `script` and start the xfce4 session. If it fails with an I/O error, start this README from the beginning.

    ```sh
    sudo script -f /dev/tty1
    startxfce4
    ```

8. Leave this ssh session open and start another one. In it, you can `export DISPLAY=:0`, and start launching GUI apps

## EL3 fibonacci service LKM

1. Upload the kernel module to PostmarketOS

    ```sh
    scp -o PubkeyAuthentication=no -o PreferredAuthentications=password ./resources/el3_demo_fib.ko ssp@172.16.42.1:
    ```

2. SSH into PostmarketOS and enter a root shell (for convenience)

   ```sh
   ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password ssp@172.16.42.1
   ssp@ssp-iphone5s: sudo -s
   ```

3. Load the kernel module and have fun!

   ```sh
   insmod el3_demo_fib.ko
   ```

### Usage

```text
# To send a new request for the N-th fibonacci number
echo $N > /proc/el3_fib_svc
# To read the results from the last request
cat /proc/el3_fib_svc
# When done
rmmod el3_demo_fib
```

For more info, about the output, etc. view [the module's README](https://gitlab.inf.ethz.ch/PRV-SHINDE/theses/ma-theses/ma-2022/221114_lyubomir_kyorovski/iphone-5s/lkm-el3-fib-svc/-/blob/master/README.md)
