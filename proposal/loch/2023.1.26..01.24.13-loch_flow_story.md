# %receipt a simple %loch app

This paper describes parts of an urbit app which will utilize the imaginary vane called %loch. This vane allows urbit to communicate with devices through vere using POSIX's `ioctl` protocol.

The app will interface with a UART based receipt printer and will print DM's that the ship receives.

The app also uses an imaginary library which provides a simple interface to the uart driver. This jams nouns into correct format that the device expects. 

## Loading of devices to Vere

The user will start vere with a list of device types and thier appropriate `urth` file name. i.e `[%uart, "/dev/ttyUSB"]`

- `vere` then watches the file and opens and mounts it when available. 
- `vere` will then notify `arvo` of any changes in the device status. This includes when the device is connected, disconnected, vere shuts down, and during bootup. 
- `arvo` will pass this information to `loch` which inturn will store the device and its current state. 
- If an agent is subscribed to status updates on the device it will forward it the state

## On installation of %receipt

When the user installs `%receipt` 

- the app will register with `%loch` that it needs the status of a `%uart` device.
- `%loch` will then accept the registration (or deny, not sure when it would do that) 
- `%loch` will then send `%receipt` a `gift` of the status.

## %receipt when notified of a new device

When %receipt is notified through %loch of a new device being connected 

- %receipt will issue an ioctl read command to get back information about the uart device.
- %receipt will then modify the settings and issue an ioctl write command to modify the uart device settings
- %receipt will save the device as active
- %receipt will print out any DMs saved in its queue

## %receipt when notified of a new DM

When %receipt is notified of a new DM being available

- %receipt will see if a uart device is active
- if %receipt finds one it will `write` the dm to the uart device
- if %receipt does not find one it will save it to its queue
