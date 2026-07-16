==========
PKE8721DAF
==========

.. tags:: chip:rtl8721dx, arch:arm, vendor:realtek

.. figure:: PKE8721DAF.png
   :scale: 50 %
   :align: center
   :alt: Realtek PKE8721DAF development board

   The PKE8721DAF development board.

The PKE8721DAF is a Realtek RTL8721Dx (dual-core Wi-Fi Host Controller)
development board. NuttX runs on the KM4 (Cortex-M33) application core as the
Wi-Fi host; the KM0 network processor runs the prebuilt vendor Wi-Fi firmware.
See the :doc:`RTL8721Dx chip documentation <../../index>` for the SoC
architecture, memory map and vendor-SDK dependency.

Features
========

* RTL8721Dx dual-core Wi-Fi Host Controller (KM4 Cortex-M33 host + KM0 NP)
* 2.4 GHz Wi-Fi (station and SoftAP)
* SPI NOR flash (shared with the NP, XIP)
* LOG-UART console

Supported in this NuttX port:

* NSH shell over the LOG-UART console
* littlefs persistent storage mounted at ``/data`` (a dedicated SPI NOR flash
  partition), backing the Wi-Fi key-value store
* Wi-Fi station and SoftAP through the ``wapi`` tool
* DHCP client (STA) and DHCP server (SoftAP)
* GPIO pins exposed as ``/dev/gpioN`` character devices (input, output and
  interrupt), driven directly on the SDK fwlib register layer
* General-purpose UARTs exposed as ``/dev/ttySN`` serial devices, driven
  directly on the SDK fwlib register layer

.. note::

   The Wi-Fi MAC/PHY is driven by the prebuilt vendor firmware on the KM0
   network processor. NuttX is the Wi-Fi host and exchanges frames with the NP
   over the on-chip IPC transport; see the chip documentation.

Buttons and LEDs
================

This NuttX port does not wire any user buttons or LEDs.

Configurations
==============

.. code:: console

   $ ./tools/configure.sh pke8721daf:<config-name>

nsh
---

Networking-enabled NSH with littlefs at ``/data`` and the ``wapi`` Wi-Fi tool.
The console is the LOG-UART at 1500000 8N1 (the rate is configured by the
bootloader and inherited by NuttX). The Wi-Fi examples below are available from
this configuration.

gpio
----

Minimal NSH with the GPIO driver and the ``gpio`` example enabled (no Wi-Fi).
The board registers three pins from its pin table (see
``boards/arm/rtl8721dx/pke8721daf/src/rtl8721dx_gpio.c``): an output at
``/dev/gpio0``, an input at ``/dev/gpio1`` and an interrupt pin at
``/dev/gpio2``. Edit that table to match a board's wiring. Exercise them with
the example::

    nsh> gpio -o 1 /dev/gpio0     # drive the output high
    nsh> gpio /dev/gpio1          # read the input
    nsh> gpio -w 1 /dev/gpio2     # wait for a rising-edge interrupt

Pins are encoded with the ``AMEBA_PA()`` / ``AMEBA_PB()`` helpers from
``arch/arm/src/common/ameba/ameba_gpio.h`` (port A/B, pin 0-31), matching the
Ameba SDK ``PinName`` layout.

uart
----

Minimal NSH with the general-purpose UART driver and the ``serialrx`` /
``serialblaster`` examples enabled (no Wi-Fi). The LOG-UART owns the console
and ``/dev/ttyS0``, so the board registers UART0 from its table (see
``boards/arm/rtl8721dx/pke8721daf/src/rtl8721dx_uart.c``) as ``/dev/ttyS1`` at
115200 8N1. Edit that table -- controller, TX/RX pads and baud -- to match a
board's wiring. The TX/RX pads use the same ``AMEBA_PA()`` / ``AMEBA_PB()``
encoding as the GPIO table; the driver muxes them to the UART function and
pulls RX high through the SDK ROM. Exercise the port with the examples (loop TX
back to RX, or wire it to a host serial adapter)::

    nsh> serialblaster    # stream a test pattern out /dev/ttyS1
    nsh> serialrx         # receive and dump bytes from /dev/ttyS1

The line format can be changed at runtime through ``tcsetattr()`` (the config
enables ``CONFIG_SERIAL_TERMIOS``). UART2 is shared with Bluetooth and is not
exposed by the driver.

Wi-Fi
=====

Station (connect to an AP)::

    nsh> wapi psk    wlan0 <password> 3
    nsh> wapi essid  wlan0 <ssid> 1
    nsh> renew wlan0

SoftAP (become an access point, with a DHCP server for clients)::

    nsh> wapi mode   wlan0 3
    nsh> wapi psk    wlan0 <password> 3
    nsh> wapi essid  wlan0 <ssid> 1
    nsh> ifconfig    wlan0 192.168.4.1 netmask 255.255.255.0
    nsh> dhcpd_start wlan0

Stop the SoftAP with ``wapi essid wlan0 <ssid> 0``.

Building and Flashing
=====================

The build auto-fetches the Realtek ``ameba-rtos`` SDK on first use and a
Realtek ``arm-none-eabi`` toolchain must be on ``PATH``; see the
:doc:`chip documentation <../../index>` for both.

.. code:: console

   $ ./tools/configure.sh pke8721daf:nsh
   $ make

This produces ``nuttx/app.bin`` (the NuttX KM4 image2), ``boot.bin`` and the
NP (KM0) image in the build directory.

After a successful build, flash via one of these methods:

**CLI (Linux/macOS)** — connect a USB-UART adapter (PL2303) and use the
built-in ``make flash`` target:

.. code:: console

   $ make flash AMEBA_PORT=/dev/ttyUSB0

The baud rate defaults to 1500000; override with ``AMEBA_BAUD`` if needed.

**GUI (Windows)** — use the Realtek AmebaImageTool (``AmebaImageTool.exe``
under ``tools/ameba/ImageTool/`` in the SDK tree) to select ``boot.bin`` and
``app.bin``.

See the `Realtek Ameba ImageTool guide
<https://aiot.realmcu.com/en/latest/tools/image_tool/index.html>`_ for the
Windows GUI tool and download-mode entry (hold the download button /
power-cycle with the ``UART_LOG_TX`` line asserted).

**Serial console** — after flashing, connect to the LOG-UART at 1500000 8N1::

    $ picocom -b 1500000 /dev/ttyUSB0

Other tools: ``screen /dev/ttyUSB0 1500000`` or ``minicom -b 1500000 -D /dev/ttyUSB0``.

License Exceptions
==================

This board depends on Realtek vendor code that is not part of NuttX and is
subject to its own license:

* The prebuilt KM0 (NP) Wi-Fi firmware image and the Realtek ``ameba-rtos``
  SDK libraries/headers linked into the image. See the SDK's own license; the
  SDK is auto-fetched and is not redistributed in the NuttX tree.
