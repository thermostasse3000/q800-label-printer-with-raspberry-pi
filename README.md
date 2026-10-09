# q800-label-printer-with-raspberry-pi
thanks to Pavel Sindler, I updated the web printer for the brother Ql800 usb label printer. I was using his manual https://evenparity.net/automatic-label-printer-with-raspberry-pi/ as template. its just the web-printer. I did the coding changes with chatgtp



This guide explains how to set up a Raspberry Pi as a browser-based label printer for the **Brother QL-800**.

It builds on the original article:
[Automatic Label Printer with Raspberry Pi — Even Parity](https://evenparity.net/automatic-label-printer-with-raspberry-pi/)

This updated guide includes instructions for a newer Python environment, automatic startup, USB permissions, and a custom black/red text selector.

## Features

- Print labels from a web browser on your local network.
- Connect a Brother QL-800 directly to the Raspberry Pi via USB.
- Select black or red text when using compatible two-colour label stock.
- Start the web application automatically at boot.
- Run the service without logging in to the desktop.

## 1. Requirements

### Hardware

- Raspberry Pi running Raspberry Pi OS.
- Brother QL-800 label printer.
- USB cable.
- Compatible 62 mm label roll.
- Network connection.

For two-colour printing, use compatible black/red/white label stock, such as Brother DK-22251.

### Software

- Python 3.
- Python virtual environment (`venv`).
- `brother_ql_web`.
- `brother_ql`.
- `fontconfig`.
- `systemd`.

## 2. Prepare the Raspberry Pi

Update the operating system:

```bash
sudo apt update
sudo apt upgrade
