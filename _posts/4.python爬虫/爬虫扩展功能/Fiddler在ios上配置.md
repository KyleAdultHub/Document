Capture Traffic from iOS Device
Configure Fiddler
Click Tools > Fiddler Options > Connections.

Click the checkbox by Allow remote computers to connect.

Allow remote computers to connect

Restart Fiddler.

Ensure your firewall allows incoming connections to the Fiddler process.

Hover over the Online indicator at the far right of the Fiddler toolbar to display the IP addresses assigned to Fiddler's machine.

Online Tooltip

Verify client iOS device can reach Fiddler by navigating in the browser to http://FiddlerMachineIP:8888. This address should return the Fiddler Echo Service page.

For iPhone: Disable the 3g/4g connection.

Set the iOS Device Proxy
Tap Settings > General > Network > Wi-Fi.

Tap the settings for the Wi-Fi network.

Tap the Manual option in the HTTP Proxy section.

In the Server box, type the IP address or hostname of your Fiddler instance.

In the Port box, type the port Fiddler is listening on (usually 8888).

Ensure the Authentication slider is set to Off.

iOS Proxy Settings

Decrypt HTTPS Traffic from iOS Devices
Download the Certificate Maker plugin for Fiddler.

Install the Certificate Maker plugin.

Restart Fiddler.

Configure the device where Fiddler is installed to trust Fiddler root certificate.

On the iOS device, go to http://ipv4.fiddler:8888/ in a browser.

From the bottom of the Fiddler Echo Service webpage, download the FiddlerRoot certificate.

Download FiddlerRoot Certificate

Open the FiddlerRoot.cer file.

Tap the Install button.

Install Profile

Tap the Install button again.

Warning

Uninstall FiddlerRoot Certificate
If you decide to uninstall the root certificate:

Tap the Settings app.

Tap General.

Scroll to Profiles.

Tap the DO_NOT_TRUST_FiddlerRoot* profile.

Tap Remove.


是否将当前网页翻译成中文 网页翻译 中英对照 