# Bootstrap Raspberry Zero W

Got a new Raspberry Zero W and needed to configure it without screen. Wrote 
down a short recipe to remember how to do it.

## Bootstrap Raspbian

- Install [Raspberian](), by using i.e. [Etcher]() on the microSD card.

- Mount the microSD card, and go to the `/boot` partition.

    - Add device tree overlay dwc2 driver to enable OTG host/gadget flipping on the OTG port.

        ```
        vi config.txt
        ```

        ```
        dtoverlay=dwc2
        ```

    - Ensure that modules, including OTG network driver, are loaded during boot.

        ```
        vi cmdline.txt
        ```

        Add `modules-load=dwc2,g_ether` after `rootwait`.
        
    - Enable SSH during first boot.

        ```
        touch ssh
        ```

- On your Linux machine, use `nm-connection-editor` and share your Internet through the `enp0sXXXX` device.

- Search for your Raspberry Zero, 10.42.0.1 is your internal interface.

    ```
    nmap -sP 10.42.0.0/24
    ```

- Log into your device, default password is `raspberry`.

    ```
    ssh -l pi 10.42.0.XXX
    ```

- Enable SSH permanently.

    ```
    sudo systemctl enable ssh
    ```

## Finish up configuration

Now finish up the configuration by using the configuration tool `raspi-config`. Set name on machine, connect to your favorite WiFi and so on.

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
