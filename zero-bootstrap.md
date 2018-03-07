# Bootstrap Raspberry Zero W


- Install [Raspberian](), by using i.e. [Etcher]() on the microSD card.

- Mount the microSD card, and go to the `/boot` partition.

    - Add device tree overlay dwc2 driver to enable OTG host/gadget flipping on the OTG port.

          vi config.txt

          dtoverlay=dwc2

    - Ensure that modules is loaded during boot.

            vi cmdline.txt

        Add `modules-load=dwc2,g_ether` after `rootwait`.
        
    - Enable SSH during boot.

            touch ssh

- Use `nm-connection-editor` and share your Internet through the `enp0sXXXX` device.

- Search for your Raspberry Zero, 10.42.0.1 is your internal interface.

        nmap -sP 10.42.0.0/24

- Log into your device, default password is `raspberry`.

        ssh -l pi 10.42.0.XXX  

[Raspberian]: https://www.raspberrypi.org/downloads/raspbian/
[Etcher]: https://etcher.io/
