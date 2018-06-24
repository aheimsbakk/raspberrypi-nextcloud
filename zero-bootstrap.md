# Bootstrap Raspberry Zero W

Got a new Raspberry Zero W and needed to configure it without screen. Wrote 
down a short recipe to remember how to do it.

Install [Raspberian](), by using i.e. [Etcher]() on the microSD card.

## On the SD card

Before you take out the microSD card and put it into the Raspberry PI Zero.

### Prepare SSH over OTG

- Mount the microSD card, and go to the `/boot` partition.

- Add device tree overlay dwc2 driver to enable OTG host/gadget flipping on the OTG port. And ensure that modules, including OTG network driver, are loaded during boot. Last but not least. Enable SSH on first boot.

    ```
    grep -q ^dtoverlay config.txt || sed -i '$ a\dtoverlay=dwc2' config.txt
    sed -i -E '/rootwait [^m]/ s/(rootwait)/\1 modules-load=dwc2,g_ether/' cmdline.txt
    touch ssh
    ```

### Prepare serial over OTG

- Mount the microSD card, and go to the `/boot` partition.

- Add device tree overlay dwc2 driver to enable OTG host/gadget flipping on the OTG port. And ensure that modules, including OTG serial driver, are loaded during boot. 

    ```
    grep -q ^dtoverlay config.txt || sed -i '$ a\dtoverlay=dwc2' config.txt
    sed -i -E '/rootwait [^m]/ s/(rootwait)/\1 modules-load=dwc2,g_serial/' cmdline.txt
    ```

- Go to the `/rootfs` partition on the microSD card.

    ```
    sudo ln -s /lib/systemd/system/getty@.service  etc/systemd/system/getty.target.wants/getty@ttyGS0.service
    ```

## First boot

Put the microSD card in the Raspberry PI Zero and connect it to the computer via the OTG port and wait for it to boot.

### Via SSH 

On your Linux machine, use `nm-connection-editor` and share your Internet through the `enp0sXXXX` device.

- Search for your Raspberry Zero, 10.42.0.1 is your internal interface.

    ```
    nmap -sP 10.42.0.0/24
    ```

- Log into your device, default password is `raspberry`.

    ```
    ssh -l pi 10.42.0.XXX
    ```

### Via serial

- On Debian, add the current user to the `dialout` group to get access to the Raspberrys serial device.

    ```
    sudo groupadd -g dialout aheimsbakk
    ```

- Connect to the Raspberry PI.

    ```
    sudo screen /dev/ttyACM0 115200
    ```

## Finish up configuration

Now finish up the configuration by using the configuration tool `raspi-config`. Set name on machine, connect to your favorite WiFi, enable SSH  and so on.

```
sudo raspi-config
```

## Secure your Zero

Don't forget to set a secure password for the `pi` user.

```
sudo passwd pi
```

## Congrats!

[Raspberian]: https://www.raspberrypi.org/downloads/raspbian/
[Etcher]: https://etcher.io/
