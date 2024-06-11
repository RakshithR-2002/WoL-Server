# WoL-Server

This is an Open Source Lightwieght Wake on Lan Server, primarily desgined to run on a Raspberry pi, but it could run on any SBC or Computer.

# About

I always found it very difficult and annoying to manage all my devices and turn them on when needed remotely.
Most Management system like PulseWay, are designed for large scale IT infrastructure and contain too many unwanted system administration tools and servcies, that can be overwhelming to a Beginner in homelab and Sys Admin.
Not to mention, that they are expensive and priced for enterprize use.
Thus, I made my own, light and easy to use Wake on Lan server, which is easy to set up with the least power usage and compute requirements as possible.

# Building a Wake-on-LAN (WoL) Server on Raspberry Pi 3B+ (C/C++)

This guide details the steps for building a WoL server on your Raspberry Pi 3B+ using C/C++. We'll break it down into clear sections with explanations and code examples.

**Prerequisites:**

* Raspberry Pi 3B+ with Raspbian (or any Debian-based) OS installed and configured
* Network connection (Ethernet recommended for Pi)
* Text editor (nano, vim, etc.)
* C/C++ compiler (gcc/g++ installed by default)
* Basic understanding of Linux commands and C/C++ programming

**Understanding Wake-on-LAN (WoL):**

WoL sends a "magic packet" over the network to a specific device. This packet contains the MAC address of the target device, triggering it to power on. We'll use UDP broadcast to send this packet.

**Steps:**

**1. Enable Wake-on-LAN in Device BIOS/UEFI:**

This step is crucial and must be done on **each** device you want to wake up. 

* Access your device's BIOS/UEFI settings during startup (usually by pressing Del, F2, F10, or Esc).
* Look for options like "Wake-on-LAN," "Power on by PME," or similar under Power Management.
* Enable the relevant options and save the changes.

**2. Determine Target Device MAC Addresses:**

* **On the target device itself:**
   * **Windows:** Open Command Prompt and type `ipconfig /all`. Find the "Physical Address" for your network adapter.
   * **Linux:** Open a terminal and use `ifconfig` or `ip link show`. Look for the "HWaddr" or "ether" address.
* **Note:** You'll need these MAC addresses for your WoL server.

**3. Install Required Libraries (on Raspberry Pi):**

We'll use standard C libraries for network programming.

```bash
sudo apt update
sudo apt install build-essential
```

**4.  C++ Code for the WoL Server:**

Create a new file (e.g., `wol_server.cpp`) using your text editor and add the following code:

```c++
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>

#define BROADCAST_ADDR "255.255.255.255"
#define WOL_PORT 9 

int main(int argc, char *argv[]) {
    if (argc < 3) {
        std::cerr << "Usage: " << argv[0] << " <MAC Address> <Broadcast IP (optional)>" << std::endl;
        return 1;
    }

    // Extract MAC address
    unsigned char mac[6];
    if (sscanf(argv[1], "%hhx:%hhx:%hhx:%hhx:%hhx:%hhx", &mac[0], &mac[1], &mac[2], &mac[3], &mac[4], &mac[5]) != 6) {
        std::cerr << "Invalid MAC address format. Use format: XX:XX:XX:XX:XX:XX" << std::endl;
        return 1;
    }

    // Broadcast IP (use default or provided argument)
    const char *broadcast_ip = (argc > 3) ? argv[2] : BROADCAST_ADDR;

    // Create UDP socket
    int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0) {
        std::cerr << "Error creating socket" << std::endl;
        return 1;
    }

    // Set socket option for broadcasting
    int broadcastEnable = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_BROADCAST, &broadcastEnable, sizeof(broadcastEnable));

    // Prepare the magic packet
    unsigned char packet[102];
    std::memset(packet, 0xFF, 6); // First 6 bytes are FF
    std::memcpy(packet + 6, mac, 6); // Repeat MAC address 16 times
    for (int i = 1; i < 16; i++) {
        std::memcpy(packet + 6 * (i + 1), mac, 6);
    }

    // Prepare sockaddr_in structure
    sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(WOL_PORT);
    inet_pton(AF_INET, broadcast_ip, &(addr.sin_addr));

    // Send the magic packet
    ssize_t bytesSent = sendto(sockfd, packet, sizeof(packet), 0, (struct sockaddr *)&addr, sizeof(addr));
    if (bytesSent != sizeof(packet)) {
        std::cerr << "Error sending magic packet" << std::endl;
    } else {
        std::cout << "Wake-on-LAN packet sent successfully!" << std::endl;
    }

    close(sockfd);
    return 0;
}
```

**5. Compile the C++ Code:**

```bash
g++ wol_server.cpp -o wol_server
```

**6. Run the WoL Server:**

```bash
sudo ./wol_server XX:XX:XX:XX:XX:XX [Broadcast IP]
```

* Replace `XX:XX:XX:XX:XX:XX` with the actual MAC address of the device you want to turn on.
* The `[Broadcast IP]` argument is optional. Use it if your network setup requires a specific broadcast address. 

**7. Test Your WoL Server:**

* Ensure the target device is powered off.
* Run the compiled `wol_server` executable with the correct MAC address.
* Your device should power on within a few seconds.

**Additional Considerations:**

* **Security:** While convenient, using broadcast for WoL on an open network can be a security concern. Consider implementing security measures for your specific network environment.
* **Error Handling:** Implement robust error handling in your code, especially for socket operations and user input.
* **GUI:** For easier use, you can create a simple graphical user interface (GUI) using libraries like Qt or GTK+.

This comprehensive guide provides a solid foundation for building your own WoL server on a Raspberry Pi. Remember to prioritize security and tailor the solution to your specific network needs. 

# Designing a UI in StreamLit

**1. Install Streamlit and other dependencies:**

   ```bash
   pip install streamlit
   pip install --upgrade streamlit
   ```

**2. Create a Python file (e.g., `wol_app.py`):**

   ```python
   import streamlit as st
   import socket
   import struct

   def wake_on_lan(mac_address):
       """Sends a magic packet to wake up a device."""
       try:
           mac_address = mac_address.replace(mac_address[2], "")
           mac_address = mac_address.replace(":", "")
           data = ''.join(['FFFFFFFFFFFF', mac_address * 16])
           send_data = b''

           for i in range(0, len(data), 2):
               send_data = b''.join([send_data, struct.pack('B', int(data[i: i + 2], 16))])
           sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
           sock.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
           sock.sendto(send_data, ('<broadcast>', 9)) 
           st.success("Wake-on-LAN packet sent successfully!")
       except Exception as e:
           st.error(f"An error occurred: {e}")

   st.title("Wake-on-LAN App")

   mac_address = st.text_input("Enter MAC Address (e.g., XX:XX:XX:XX:XX:XX):")
   if st.button("Wake Up Device"):
       if mac_address:
           wake_on_lan(mac_address.upper())
       else:
           st.warning("Please enter a MAC address.")
   ```

**3. Run the Streamlit app:**

   ```bash
   streamlit run wol_app.py 
   ```

**Explanation:**

- **Import Necessary Libraries:**
   - `streamlit` is used for building the web app.
   - `socket` provides networking capabilities.
   - `struct` is used for packing the magic packet data.
- **`wake_on_lan(mac_address)` Function:**
   - Takes the MAC address as input.
   - Constructs the magic packet payload (6 bytes of `FF` followed by 16 repetitions of the MAC address).
   - Creates a UDP socket and sets the `SO_BROADCAST` option to enable broadcasting.
   - Sends the magic packet to the broadcast address (`255.255.255.255`) on port 9 (the WoL port).
- **Streamlit App:**
   - `st.title()` sets the title of the app.
   - `st.text_input()` creates an input field for the MAC address.
   - `st.button()` creates the "Wake Up Device" button.
   - The code within the `if st.button()` block is executed when the button is clicked:
     - It retrieves the MAC address from the input field.
     - If a MAC address is provided, it calls the `wake_on_lan()` function; otherwise, it displays a warning message.

Now, when you run the Streamlit app and enter a valid MAC address, clicking the "Wake Up Device" button will send the WoL magic packet, potentially turning on the corresponding device.

