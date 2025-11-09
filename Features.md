# BreezyDesktop Features

This document outlines the implementation details of key features in the BreezyDesktop application.

## Screen x y offsets

This feature allows the user to adjust the viewport's horizontal and vertical offsets.

* **GNOME Shell Extension:** The core logic for applying the offsets is in [`gnome/src/extension.js`](./gnome/src/extension.js). The `_effect_enable` function reads the `viewport-offset-x` and `viewport-offset-y` settings and passes them to the `VirtualDisplaysActor`.
* **UI:** The user interface for controlling the offsets is defined in [`ui/src/gtk/connected-device.ui`](./ui/src/gtk/connected-device.ui). This file contains the GtkScale and GtkAdjustment widgets for both the x and y offsets.
* **UI Logic:** The Python code that manages the UI and binds the settings to the widgets is located in [`ui/src/connecteddevice.py`](./ui/src/connecteddevice.py).
* **Settings Schema:** The GSettings schema, which defines the `viewport-offset-x` and `viewport-offset-y` keys, is located in [`ui/data/com.xronlinux.BreezyDesktop.gschema.xml`](./ui/data/com.xronlinux.BreezyDesktop.gschema.xml).

## Focus Modes and Zoom on Focus

This feature allows the user to choose how the focused monitor is determined, with three exclusive modes: "None," "Gyroscope," and "Keyboard." Additionally, the user can enable or disable a "zoom on focus" feature, which makes the focused display appear closer.

* **GNOME Shell Extension:**
    * **Configuration:** The `focus-mode` and `zoom-on-focus-enabled` settings are defined in the GSettings schema at [`ui/data/com.xronlinux.BreezyDesktop.gschema.xml`](./ui/data/com.xronlinux.BreezyDesktop.gschema.xml).
    * **UI:** The UI for selecting the focus mode is a dropdown menu in the GTK settings panel, defined in [`ui/src/gtk/connected-device.ui`](./ui/src/gtk/connected-device.ui). The "Zoom on focus" feature is controlled by a switch in the same file. The corresponding Python logic is in [`ui/src/connecteddevice.py`](./ui/src/connecteddevice.py).
    * **Core Logic:** The [`gnome/src/virtualdisplaysactor.js`](./gnome/src/virtualdisplaysactor.js) file contains the logic for handling the different focus modes and the zoom-on-focus feature. The gyroscope-based focus is only active when `focus-mode` is set to `'gyroscope'`, and the keyboard-based focus is only active when it's set to `'keyboard'`. The zoom effect is applied when `zoom-on-focus-enabled` is true and a monitor is focused.
* **KWin Effect:**
    * **Configuration:** The `FocusMode` and `ZoomOnFocusEnabled` settings are defined in the KCM configuration file at [`kwin/src/breezydesktopconfig.kcfg`](./kwin/src/breezydesktopconfig.kcfg).
    * **UI:** The KCM UI, defined in [`kwin/src/kcm/breezydesktopeffectkcm.ui`](./kwin/src/kcm/breezydesktopeffectkcm.ui), uses a `QComboBox` to allow the user to select the focus mode and a `QCheckBox` for the "Zoom on focus" feature.
    * **Core Logic:** The `BreezyDesktopEffect::reconfigure` method in [`kwin/src/breezydesktopeffect.cpp`](./kwin/src/breezydesktopeffect.cpp) reads the `focusMode` and `zoomOnFocusEnabled` from the config, and the `focusNext()` and `focusPrevious()` methods are conditionally executed based on the value of `m_focusMode`. The zoom effect is applied by the QML code, which reads the `zoomOnFocusEnabled` property.

## Keyboard shortcuts

This feature allows the user to control various aspects of the application using keyboard shortcuts.

* **GNOME Shell Extension:** The [`gnome/src/extension.js`](./gnome/src/extension.js) file is responsible for registering and handling keyboard shortcuts. The `_add_settings_keybinding` function is used to bind keyboard shortcuts to specific actions.
* **Shortcut Definitions:** The available shortcuts are defined as GSettings keys in [`ui/data/com.xronlinux.BreezyDesktop.gschema.xml`](./ui/data/com.xronlinux.BreezyDesktop.gschema.xml). These include shortcuts for recentering the display, toggling the display distance, toggling follow mode, moving the cursor to the focused display, and focusing the next/previous display.
* **UI:** The [`ui/src/shortcutdialog.py`](./ui/src/shortcutdialog.py) and [`ui/src/gtk/shortcut-dialog.ui`](./ui/src/gtk/shortcut-dialog.ui) files provide a dialog for viewing and editing the keyboard shortcuts.

## Processing of glasses gyroscope data

The processing of the gyroscope data is handled by the driver, which communicates with the application through a shared memory file.

* **Driver:** The driver is responsible for reading the raw data from the glasses, performing sensor fusion and calibration, and writing the processed data to a shared memory file.
* **Shared Memory:** The [`gnome/src/devicedatastream.js`](./gnome/src/devicedatastream.js) file reads the processed gyroscope data from the `/dev/shm/breezy_desktop_imu` shared memory file.
* **Data Stream:** The `DeviceDataStream` class parses the data from the shared memory file and makes it available to the rest of the application through the `imu_snapshots` property. The `POSE_ORIENTATION` field contains the quaternion representing the orientation of the glasses.
* **IPC:** The `XRDriverIPC` class, with implementations in both C++ ([`kwin/src/xrdriveripc`](./kwin/src/xrdriveripc)) and Python ([`ui/src/xrdriveripc.py`](./ui/src/xrdriveripc.py)), is used for general communication with the driver, but the high-frequency gyroscope data is transferred through the shared memory file for performance reasons.

## OpenTrack Integration

The driver includes two plugins for bi-directional communication with OpenTrack over UDP. This allows for using OpenTrack as a head-tracking source for the virtual displays, and for sending the virtual display's orientation to other applications via OpenTrack.

*   **Listener Plugin:** The [`modules/XRLinuxDriver/src/plugins/opentrack_listener.c`](./modules/XRLinuxDriver/src/plugins/opentrack_listener.c) file implements a UDP listener that receives data from OpenTrack. When data is received, the plugin creates a synthetic IMU device, allowing the rest of the driver to treat OpenTrack as a standard data source. The plugin is responsible for converting OpenTrack's EUS (East, Up, South) coordinate system to the driver's NWU (North, West, Up) system.

*   **Source Plugin:** The [`modules/XRLinuxDriver/src/plugins/opentrack_source.c`](./modules/XRLinuxDriver/src/plugins/opentrack_source.c) file implements a UDP sender that transmits pose data to OpenTrack. This plugin is activated when the `external_mode` is set to `"opentrack"` in the driver's configuration. It takes the driver's pose data, converts it to the EUS coordinate system, and sends it to the configured OpenTrack IP address and port.
