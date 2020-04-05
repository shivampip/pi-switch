# pi-switch

## Code 1
* Python script to control bluetooth.
```python
from btcom import *  # TigerJython
from btpycom import *  # Standard-Python (PC, Raspi, ...)

def onStateChanged(state, msg):
    global reply
    if state == "CONNECTING":
        print "Connecting", msg
    elif state == "CONNECTION_FAILED":
        print "Connection failed", msg
    elif state == "CONNECTED":
        print "Connected", msg
    elif state == "DISCONNECTED":
        print "Disconnected", msg
    elif state == "MESSAGE":
        print "Message", msg
        reply = msg
       
serviceName = "EchoServer"
print "Performing search for service name", serviceName
client = BTClient(stateChanged = onStateChanged)
serverInfo = client.findService(serviceName, 20)
if serverInfo == None:
    print "Service search failed"
else:
    print "Got server info", serverInfo
    if client.connect(serverInfo, 20):
        for n in range(0, 101):
            client.sendMessage(str(n))
            reply = ""
            while reply == "":
                time.sleep(0.001)
        client.disconnect()
```

* Python script to control Wifi

```Python
#!/usr/bin/env python
#Run.py
import os
from bluetooth import *
from wifi import Cell, Scheme
import subprocess
import time
wpa_supplicant_conf = "/etc/wpa_supplicant/wpa_supplicant.conf"
sudo_mode = "sudo "
def wifi_connect(ssid, psk):
    # write wifi config to file
    cmd = 'wpa_passphrase {ssid} {psk} | sudo tee -a {conf} > /dev/null'.format(
            ssid=str(ssid).replace('!', '\!'),
            psk=str(psk).replace('!', '\!'),
            conf=wpa_supplicant_conf
        )
    cmd_result = ""
    cmd_result = os.system(cmd)
    print cmd + " - " + str(cmd_result)
    # reconfigure wifi
    cmd = sudo_mode + 'wpa_cli -i wlan0 reconfigure'
    cmd_result = os.system(cmd)
    print cmd + " - " + str(cmd_result)
    time.sleep(10)
    cmd = 'iwconfig wlan0'
    cmd_result = os.system(cmd)
    print cmd + " - " + str(cmd_result)
    cmd = 'ifconfig wlan0'
    cmd_result = os.system(cmd)
    print cmd + " - " + str(cmd_result)
    p = subprocess.Popen(['hostname', '-I'], stdout=subprocess.PIPE,
                            stderr=subprocess.PIPE)
    out, err = p.communicate()
    if out:
        ip_address = out
    else:
        ip_address = "<Not Set>"
    return ip_address
def ssid_discovered():
    Cells = Cell.all('wlan0')
    wifi_info = 'Found ssid : \n'
    for current in range(len(Cells)):
        wifi_info +=  Cells[current].ssid + "\n"
    wifi_info+="!"
    print wifi_info
    return wifi_info
def handle_client(client_sock) :
    # get ssid
    client_sock.send(ssid_discovered())
    print "Waiting for SSID..."
    ssid = client_sock.recv(1024)
    if ssid == '' :
        return
    print "ssid received"
    print ssid
    # get psk
    client_sock.send("waiting-psk!")
    print "Waiting for PSK..."
    psk = client_sock.recv(1024)
    if psk == '' :
        return
    print "psk received"
    print psk
    ip_address = wifi_connect(ssid, psk)
    print "ip address: " + ip_address
    client_sock.send("ip-address:" + ip_address + "!")
    return
try:
    while True:
        server_sock=BluetoothSocket( RFCOMM )
        server_sock.bind(("",PORT_ANY))
        server_sock.listen(1)
        port = server_sock.getsockname()[1]
        uuid = "815425a5-bfac-47bf-9321-c5ff980b5e11"
        advertise_service( server_sock, "RPi Wifi config",
                           service_id = uuid,
                           service_classes = [ uuid, SERIAL_PORT_CLASS ],
                           profiles = [ SERIAL_PORT_PROFILE ])
        print "Waiting for connection on RFCOMM channel %d" % port
        client_sock, client_info = server_sock.accept()
        print "Accepted connection from ", client_info
        handle_client(client_sock)
        client_sock.close()
        server_sock.close()
        # finished config
        print 'Finished configuration\n'
except (KeyboardInterrupt, SystemExit):
    print '\nExiting\n'
```

## Code 2

```Python
#!/usr/bin/env python

import os
from bluetooth import *
from wifi import Cell, Scheme
import subprocess
import time




wpa_supplicant_conf = "/etc/wpa_supplicant/wpa_supplicant.conf"
sudo_mode = "sudo "


def wifi_connect(ssid, psk):
    # write wifi config to file
    f = open('wifi.conf', 'w')
    f.write('country=GB\n')
    f.write('ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev\n')
    f.write('update_config=1\n')
    f.write('\n')
    f.write('network={\n')
    f.write('    ssid="' + ssid + '"\n')
    f.write('    psk="' + psk + '"\n')
    f.write('}\n')
    f.close()

    cmd = 'mv wifi.conf ' + wpa_supplicant_conf
    cmd_result = ""
    cmd_result = os.system(cmd)
    print cmd + " - " + str(cmd_result)


    # restart wifi adapter
    cmd = sudo_mode + 'ifdown wlan0'
    cmd_result = os.system(cmd)
    print cmd + " - " + str(cmd_result)

    time.sleep(2)

    cmd = sudo_mode + 'ifup wlan0'
    cmd_result = os.system(cmd)
    print cmd + " - " + str(cmd_result)

    time.sleep(10)

    cmd = 'iwconfig wlan0'
    cmd_result = os.system(cmd)
    print cmd + " - " + str(cmd_result)

    cmd = 'ifconfig wlan0'
    cmd_result = os.system(cmd)
    print cmd + " - " + str(cmd_result)

    p = subprocess.Popen(['ifconfig', 'wlan0'], stdout=subprocess.PIPE,
                            stderr=subprocess.PIPE)

    out, err = p.communicate()

    ip_address = "<Not Set>"

    for l in out.split('\n'):
        if l.strip().startswith("inet addr:"):
            ip_address = l.strip().split(' ')[1].split(':')[1]

    return ip_address



def ssid_discovered():
    Cells = Cell.all('wlan0')

    wifi_info = 'Found ssid : \n'

    for current in range(len(Cells)):
        wifi_info +=  Cells[current].ssid + "\n"


    wifi_info+="!"

    print wifi_info
    return wifi_info


def handle_client(client_sock) :
    # get ssid
    client_sock.send(ssid_discovered())
    print "Waiting for SSID..."


    ssid = client_sock.recv(1024)
    if ssid == '' :
        return

    print "ssid received"
    print ssid

    # get psk
    client_sock.send("waiting-psk!")
    print "Waiting for PSK..."


    psk = client_sock.recv(1024)
    if psk == '' :
        return

    print "psk received"

    print psk

    ip_address = wifi_connect(ssid, psk)

    print "ip address: " + ip_address

    client_sock.send("ip-addres:" + ip_address + "!")

    return



try:
    while True:
        server_sock=BluetoothSocket( RFCOMM )
        server_sock.bind(("",PORT_ANY))
        server_sock.listen(1)

        port = server_sock.getsockname()[1]

        uuid = "815425a5-bfac-47bf-9321-c5ff980b5e11"

        advertise_service( server_sock, "RPi Wifi config",
                           service_id = uuid,
                           service_classes = [ uuid, SERIAL_PORT_CLASS ],
                           profiles = [ SERIAL_PORT_PROFILE ])


        print "Waiting for connection on RFCOMM channel %d" % port

        client_sock, client_info = server_sock.accept()
        print "Accepted connection from ", client_info

        handle_client(client_sock)

        client_sock.close()
        server_sock.close()

        # finished config
        print 'Finished configuration\n'


except (KeyboardInterrupt, SystemExit):
    print '\nExiting\n'
```
