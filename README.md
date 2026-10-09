# q800-label-printer-with-raspberry-pi
thanks to Pavel Sindler, I updated the web printer for the brother Ql800 usb label printer. I was using his manual https://evenparity.net/automatic-label-printer-with-raspberry-pi/ as template. its just the web-printer. I did the coding changes with chatgtp



# Raspberry Pi Automatic Label Printer

This guide explains how to set up a Raspberry Pi to run a Brother QL-800 label printer through a web interface using [`brother_ql_web`](https://github.com/).

The setup includes:

- A web-based label designer.
- Automatic startup through `systemd`.
- USB connection to the Brother QL-800.
- Support for black-only and black/red printing.
- A dropdown menu to choose the text color in the label designer.

> **Note:** Replace `username` with your actual Raspberry Pi username when applying these instructions. Replace `<RASPBERRY-PI-IP>` with your Raspberry Pi's IP address.

---

## 1. Requirements

You will need:

- A Raspberry Pi running Raspberry Pi OS.
- A Brother QL-800 label printer.
- A compatible label roll, such as DK-22251 black/red continuous-length tape.
- A USB cable connecting the printer to the Raspberry Pi.
- Network access to the Raspberry Pi.

Connect the printer to the Raspberry Pi and switch it on.

---

## 2. Update Raspberry Pi OS

Update the package lists and installed packages:

```bash
sudo apt update
sudo apt upgrade -y
```

Install the required packages:

```bash
sudo apt install -y \
    python3 \
    python3-pip \
    python3-venv \
    python3-dev \
    git \
    cups \
    build-essential
```

---

## 3. Create a Python virtual environment

Create a Python virtual environment in the user's home directory:

```bash
cd /home/username
python3 -m venv venv
```

Activate the environment:

```bash
source /home/username/venv/bin/activate
```

Upgrade pip:

```bash
pip install --upgrade pip
```

Install the web application:

```bash
pip install brother_ql_web
```

Verify that the package is installed:

```bash
python -m brother_ql_web --help
```

If the help command is unavailable or the package version uses different command-line options, consult the installation instructions for the version you installed.

---

## 4. Check the USB printer connection

Connect the Brother QL-800 to the Raspberry Pi using USB.

Check whether the printer is detected:

```bash
lsusb
```

You should see a USB device corresponding to Brother.

Check whether the printer device exists:

```bash
ls -l /dev/usb/lp*
```

The printer may appear as:

```text
/dev/usb/lp0
```

The exact device path can vary depending on the connected USB devices and system configuration.

If `/dev/usb/lp0` does not exist, check the USB connection and the printer's power. You may also need to investigate the appropriate kernel modules and USB device permissions.

---

## 5. Configure printer permissions

The web application needs permission to access the printer.

If the printer is available through `/dev/usb/lp0`, inspect its ownership and permissions:

```bash
ls -l /dev/usb/lp0
```

On systems where the device is owned by the `lp` group, add the Raspberry Pi user to that group:

```bash
sudo usermod -aG lp username
```

Log out and log back in, or reboot, so the new group membership takes effect.

Verify group membership:

```bash
groups username
```

The output should include `lp`.

> **Security note:** Access to printer device files can allow direct control of the printer. Grant only the permissions needed by the account running the application.

---

## 6. Create the configuration file

Create a configuration file in the user's home directory:

```bash
nano /home/username/config.json
```

Use the following configuration as a starting point:

```json
{
    " BROTHER_QL_WEB_PRINTER": "file:///dev/usb/lp0",
    "BROTHER_QL_WEB_LABEL": "62red"
}
```

**Important:** The exact configuration keys supported by `brother_ql_web` depend on the installed version. Check the package's configuration documentation and use the keys expected by your version. In particular, ensure that the printer setting is spelled exactly as required; do not include a leading space in a JSON key.

For versions that use the standard lowercase configuration keys, the configuration should instead look like this:

```json
{
    "printer": "file:///dev/usb/lp0",
    "label": "62red"
}
```

Save the file and exit the editor.

The `62red` label setting is intended to select the compatible 62 mm black/red label mode. Use the label identifier appropriate to your installed `brother_ql_web` version and the media loaded in the printer.

### Check the configuration file

Validate the JSON syntax:

```bash
python -m json.tool /home/username/config.json
```

If the command reports a JSON parsing error, correct the file before continuing.

---

## 7. Test the web application manually

Activate the virtual environment:

```bash
source /home/username/venv/bin/activate
```

Start the web application with the configuration file:

```bash
python -m brother_ql_web \
    --configuration /home/username/config.json
```

If the installed version uses a different command-line syntax, follow its help output:

```bash
python -m brother_ql_web --help
```

From another device on the same network, open:

```text
http://<RASPBERRY-PI-IP>:8080/labeldesigner
```

For example, if the Raspberry Pi's IP address is `192.168.1.50`, use:

```text
http://192.168.1.50:8080/labeldesigner
```

To find the Raspberry Pi's IP address, run:

```bash
hostname -I
```

Confirm that the label designer loads and that the printer can print a test label.

Stop the manually started application with `Ctrl+C` before configuring the service.

---

## 8. Configure automatic startup with systemd

Create a systemd service:

```bash
sudo nano /etc/systemd/system/brother-ql-web.service
```

Add the following:

```ini
[Unit]
Description=Brother QL Web Label Printer
After=network.target

[Service]
Type=simple
User=username
WorkingDirectory=/home/username
ExecStart=/home/username/venv/bin/python -m brother_ql_web --configuration /home/username/config.json
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Save the file and exit.

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Enable the service so it starts automatically at boot:

```bash
sudo systemctl enable brother-ql-web.service
```

Start the service:

```bash
sudo systemctl start brother-ql-web.service
```

Check its status:

```bash
sudo systemctl status brother-ql-web.service
```

If the service starts successfully, the web application should now run in the background.

### View service logs

To inspect the latest logs:

```bash
sudo journalctl -u brother-ql-web.service -n 100 --no-pager
```

To follow the logs live:

```bash
sudo journalctl -u brother-ql-web.service -f
```

---

## 9. Add a black/red text color dropdown

The Brother QL-800 supports black and red printing when compatible black/red media is installed and the printer is configured for that media.

The web application must generate an image with the correct colors, and the printer must be configured to use the appropriate black/red printing mode.

The following changes add a text-color dropdown to the label designer.

> **Important:** These instructions modify files inside the installed Python package. Package upgrades may overwrite the changes. Back up each file before editing it.
>
> The exact file paths and code structure can vary between package versions. If the paths below do not exist, locate the corresponding files in your virtual environment before making changes.

### 9.1 Locate the installed package

Run:

```bash
source /home/username/venv/bin/activate
python -c "import brother_ql_web; print(brother_ql_web.__file__)"
```

This prints the location of the installed package.

Depending on the version, the relevant files may include:

```text
brother_ql_web/
├── labels.py
├── web.py
└── views/
    └── labeldesigner.jinja2
```

Locate the actual files before editing them.

### 9.2 Back up the files

For each file you plan to modify, create a backup first.

For example:

```bash
cp /path/to/brother_ql_web/labels.py \
   /path/to/brother_ql_web/labels.py.bak

cp /path/to/brother_ql_web/web.py \
   /path/to/brother_ql_web/web.py.bak

cp /path/to/brother_ql_web/views/labeldesigner.jinja2 \
   /path/to/brother_ql_web/views/labeldesigner.jinja2.bak
```

Replace `/path/to/brother_ql_web/` with the actual directory reported by your installation.

### 9.3 Modify `labels.py`

Open the file:

```bash
nano /path/to/brother_ql_web/labels.py
```

Find the `LabelParameters` class.

Add a color parameter with a default of black:

```python
color: str = "black"
```

For example, if the class is a dataclass, the new field may look like this:

```python
@dataclass
class LabelParameters:
    # Keep the existing fields here.

    color: str = "black"
```

Do not remove the existing fields or replace the entire class with this abbreviated example.

Next, locate the code that determines the text's fill color. Modify it so the selected color is used.

For example:

```python
@property
def fill_color(self):
    if self.color == "red":
        return (255, 0, 0)

    return (0, 0, 0)
```

This returns red for the `red` option and black otherwise.

**Important:** If your existing code uses a different structure, adapt the logic to it rather than replacing unrelated code. The RGB values and rendering behavior must be compatible with the image-processing code used by your installed version.

### 9.4 Modify `web.py`

Open the file:

```bash
nano /path/to/brother_ql_web/web.py
```

Locate the section that reads form parameters and constructs the label parameters.

Add the selected color to the dictionary or parameter object passed to the label renderer:

```python
"color": d.get("color", "black"),
```

For example, if the existing code constructs parameters from a dictionary, the relevant part may resemble:

```python
parameters = {
    # Keep the existing parameters here.
    "color": d.get("color", "black"),
}
```

This is an illustrative snippet. Add the new entry to the existing parameter construction code instead of replacing the entire dictionary.

The default value ensures that labels remain black if the form does not submit a color.

### 9.5 Modify `labeldesigner.jinja2`

Open the template:

```bash
nano /path/to/brother_ql_web/views/labeldesigner.jinja2
```

Find the controls used to configure the label text.

Add a dropdown for the text color:

```html
<label for="textColor">Text color:</label>

<select id="textColor" name="color">
    <option value="black" selected>Black</option>
    <option value="red">Red</option>
</select>
```

Place the dropdown in the label designer alongside the existing text settings.

Next, find the JavaScript that gathers form values and submits them to the application.

Add code to read the selected color:

```javascript
const color = document.getElementById('textColor').value;
```

When creating the request's form data, include:

```javascript
data.append('color', color);
```

For example, if the existing JavaScript uses a `FormData` object named `data`, the new entry should be added alongside the existing fields:

```javascript
const color = document.getElementById('textColor').value;
data.append('color', color);
```

Do not create a second, unrelated submission handler if the page already has one. Integrate the new field into the existing handler.

### 9.6 Preserve black/red printer mode

The printer must be configured for compatible black/red media.

If your application uses a label parameter similar to:

```python
red: bool = "red" in parameters.label_size
```

retain this behavior if it is how your installed version determines whether red printing is enabled.

The distinction is important:

- The **text-color dropdown** chooses the color used to render the text.
- The **label media setting** enables the printer's black/red printing mode.

Changing the text color alone does not necessarily enable red printing on the printer.

Also note that black/red thermal labels generally use separate black and red printing passes or other media-specific processing. Ensure that the application's image-processing and printer-driver code supports the media you are using.

---

## 10. Restart the service

After modifying the application files, restart the service:

```bash
sudo systemctl restart brother-ql-web.service
```

Check the status:

```bash
sudo systemctl status brother-ql-web.service
```

Review the logs if the service fails:

```bash
sudo journalctl -u brother-ql-web.service -n 100 --no-pager
```

Open the label designer again:

```text
http://<RASPBERRY-PI-IP>:8080/labeldesigner
```

The text-color dropdown should appear if the template changes were applied successfully.

Test both options:

1. Select **Black** and print a test label.
2. Select **Red** and print another test label.
3. Confirm that the printer uses the expected colors.
4. Confirm that black text still works as expected.

If red text prints black, check the media setting, image color handling, and whether the installed driver supports the selected black/red label type.

---

## 11. Useful service commands

### Start the service

```bash
sudo systemctl start brother-ql-web.service
```

### Stop the service

```bash
sudo systemctl stop brother-ql-web.service
```

### Restart the service

```bash
sudo systemctl restart brother-ql-web.service
```

### Check the status

```bash
sudo systemctl status brother-ql-web.service
```

### Enable automatic startup

```bash
sudo systemctl enable brother-ql-web.service
```

### Disable automatic startup

```bash
sudo systemctl disable brother-ql-web.service
```

### View the logs

```bash
sudo journalctl -u brother-ql-web.service -n 100 --no-pager
```

---

## 12. Troubleshooting

### The web interface does not load

Check whether the service is running:

```bash
sudo systemctl status brother-ql-web.service
```

Review the logs:

```bash
sudo journalctl -u brother-ql-web.service -n 100 --no-pager
```

Check the Raspberry Pi's IP address:

```bash
hostname -I
```

Confirm that the URL uses the correct IP address and port:

```text
http://<RASPBERRY-PI-IP>:8080/labeldesigner
```

Also check that the application is listening on a network-accessible interface, rather than only on `127.0.0.1`.

### The printer is not detected

Check the USB devices:

```bash
lsusb
```

Check the printer device:

```bash
ls -l /dev/usb/lp*
```

Make sure the printer is switched on and connected directly to the Raspberry Pi.

### The application cannot access the printer

Check the device permissions:

```bash
ls -l /dev/usb/lp0
```

Check the user's groups:

```bash
groups username
```

If appropriate for your system, add the user to the `lp` group:

```bash
sudo usermod -aG lp username
```

Then log out and log back in, or reboot.

### The service fails to start

Inspect the logs:

```bash
sudo journalctl -u brother-ql-web.service -n 100 --no-pager
```

Verify the paths in the service file:

```ini
User=username
WorkingDirectory=/home/username
ExecStart=/home/username/venv/bin/python -m brother_ql_web --configuration /home/username/config.json
```

Confirm that the virtual environment and configuration file exist:

```bash
ls -l /home/username/venv/bin/python
ls -l /home/username/config.json
```

Check that the configuration file is valid JSON:

```bash
python -m json.tool /home/username/config.json
```

### The color dropdown does not appear

Check that the correct template file was modified.

Restart the service:

```bash
sudo systemctl restart brother-ql-web.service
```

Refresh the browser page. If necessary, perform a hard refresh to avoid using cached resources.

### Red text does not print red

Check the following:

- The printer has compatible black/red media installed.
- The configured label type enables black/red mode.
- The selected color is submitted by the browser.
- The server reads the color value.
- The renderer uses the selected color.
- The image-processing code and printer driver preserve the intended red channel.

Black/red media may require specific image and driver handling. A simple RGB change by itself is not sufficient if the printing pipeline converts the image to black-only output.

---

## 13. Updating the application

If you upgrade `brother_ql_web`, your custom modifications may be overwritten.

Before upgrading:

1. Back up the modified files.
2. Record the changes made to `labels.py`, `web.py`, and `labeldesigner.jinja2`.
3. Upgrade the package.
4. Reapply the changes carefully if the new version requires them.
5. Test both black and red printing.

For example, back up the modified files in a separate directory:

```bash
mkdir -p /home/username/brother-ql-web-backup
```

Copy the files you changed into that directory, adjusting the source paths to match your installation.

Avoid blindly overwriting files from a newer package version with old copies, as this can break compatibility.

---

## 14. Final checklist

Before considering the setup complete, verify the following:

- [ ] Raspberry Pi OS is updated.
- [ ] Python virtual environment exists.
- [ ] `brother_ql_web` is installed.
- [ ] Brother QL-800 is connected by USB.
- [ ] The printer device is accessible.
- [ ] The configuration file is valid.
- [ ] The web interface loads from another device.
- [ ] A black test label prints correctly.
- [ ] A red test label prints correctly, if using black/red media.
- [ ] The systemd service starts successfully.
- [ ] The service starts automatically after reboot.
- [ ] Custom changes are backed up.

---

## References

- Original setup guide: https://evenparity.net/automatic-label-printer-with-raspberry-pi/
- Raspberry Pi documentation: https://www.raspberrypi.com/documentation/
- systemd service documentation: https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html

