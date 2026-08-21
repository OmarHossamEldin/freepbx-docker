# FreePBX 17 + Asterisk 22 + Softphone Testing Guide

## Goal

The goal is to build this local development environment:

```text
                    Ubuntu 24.10
                         │
                         ▼
                      Docker
                         │
                         ▼
                ┌─────────────────┐
                │   FreePBX 17    │
                │                 │
                │   Asterisk 22   │
                │     PJSIP       │
                └───────┬─────────┘
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
         Extension 1001      Extension 1002
              │                   │
              ▼                   ▼
          Softphone 1          Softphone 2
```



Wait for the initial installation to finish.

The first installation can take considerably longer than subsequent starts because the image and FreePBX components need to be prepared.

---

Make sure the FreePBX container and database container are running.

Also run:

```bash
sudo docker ps
```

You should see the running FreePBX environment.

---

# 5. Find Your Ubuntu IP Address

Run:

```bash
hostname -I
```

Example:

```text
192.168.1.50
```

We'll call this:

```text
PBX_IP = 192.168.1.50
```

Your actual IP will probably be different.

### Important

Do **not** use:

```text
172.x.x.x
```

or the Docker container IP for your softphone.

Use the **Ubuntu host/LAN IP**.

Example:

```text
192.168.1.50
```

---

# 6. Open FreePBX

From your browser:

```text
http://192.168.1.50
```

Replace `192.168.1.50` with your actual Ubuntu IP.

You should reach the FreePBX web interface.


# 1. Create Extension 1001

In FreePBX:

```text
Applications
    ↓
Extensions
    ↓
Add New
    ↓
Add New PJSIP Extension
```

Configure:

```text
User Extension:
1001

Display Name:
Test Phone 1
```

Set a strong secret/password.

For example:

```text
SIP Secret:
CHANGE_THIS_TO_A_STRONG_PASSWORD
```

Do not use this example password in production.

Click:

```text
Submit
```

Then:

```text
Apply Config
```

---

# 2. Create Extension 1002

Create another PJSIP extension:

```text
User Extension:
1002

Display Name:
Test Phone 2
```

Set another password.

Example:

```text
SIP Secret:
CHANGE_THIS_TO_ANOTHER_STRONG_PASSWORD
```

Click:

```text
Submit
```

Then:

```text
Apply Config
```

Now you have:

```text
1001
1002
```

---

# 3. Verify the Extensions from Asterisk

Open the Asterisk CLI:

```bash
sudo docker compose exec freepbx asterisk -rvvv
```

Run:

```text
pjsip show endpoints
```

You should see your endpoints.

You can also check a specific endpoint:

```text
pjsip show endpoint 1001
```

and:

```text
pjsip show endpoint 1002
```

Exit:

```text
exit
```

---

# 4. Configure Softphone 1

You can use Linphone, MicroSIP, Zoiper, or another SIP client.

For the first phone, use:

```text
SIP Server:
192.168.1.50

Username:
1001

Password:
YOUR_1001_SECRET

Port:
5060

Transport:
UDP
```

If your softphone asks for a SIP address:

```text
sip:1001@192.168.1.50
```

Do not use:

```text
http://192.168.1.50
```

The SIP account is not an HTTP account.

---

# 5. Configure Softphone 2

Configure the second softphone:

```text
SIP Server:
192.168.1.50

Username:
1002

Password:
YOUR_1002_SECRET

Port:
5060

Transport:
UDP
```

SIP URI:

```text
sip:1002@192.168.1.50
```

---

# 6. Check SIP Registration

On Ubuntu:

```bash
sudo docker compose exec freepbx asterisk -rx "pjsip show contacts"
```

You want to see contacts associated with:

```text
1001
1002
```

You can also check:

```bash
sudo docker compose exec freepbx asterisk -rx "pjsip show endpoints"
```

The endpoints should show as available/registered once the softphones successfully register.

---

# 7. Test Extension 1001

On softphone 1, verify that the account is registered.

It should show something similar to:

```text
Registered
```

The account is:

```text
1001
```

---

# 8. Test Extension 1002

On softphone 2, verify:

```text
Registered
```

The account is:

```text
1002
```

At this point:

```text
Softphone 1
    │
    │ REGISTER
    ▼
Asterisk
    │
    │ REGISTER
    ▼
Softphone 2
```

---

# 9. Make the First Call

From softphone 1:

```text
Dial: 1002
```

The expected SIP flow is:

```text
1001
 │
 │ INVITE
 ▼
Asterisk
 │
 │ INVITE
 ▼
1002
 │
 │ 180 Ringing
 ▼
Asterisk
 │
 │
 │ 200 OK
 ▼
1001
 │
 │ ACK
 ▼
Call connected
```

You should now have two-way audio.

---

# 10. Understand SIP vs RTP

This distinction is very important for your VoIP development.

### SIP

SIP controls the call:

```text
REGISTER
INVITE
180 Ringing
200 OK
ACK
BYE
```

### RTP

RTP carries the audio:

```text
Voice
  │
  ▼
RTP packets
  │
  ▼
Other phone
```

Therefore, you can have:

```text
SIP works
+
RTP works
=
Successful call with audio
```

Or:

```text
SIP works
+
RTP broken
=
Call connects but no audio
```

---

# 11. Check RTP Configuration

Run:

```bash
sudo docker compose exec freepbx asterisk -rx "rtp show settings"
```

Check the RTP port range.

Your Docker configuration must expose the configured RTP UDP range.

For example:

```text
10000-20000/UDP
```

The exact range should match the configuration used by your Docker FreePBX project.

---

# 12. SIP Debugging

If the softphone doesn't register, enter Asterisk:

```bash
sudo docker compose exec freepbx asterisk -rvvv
```

Enable SIP logging:

```text
pjsip set logger on
```

Now try registering the softphone.

You should see SIP traffic such as:

```text
REGISTER
401 Unauthorized
REGISTER
200 OK
```

A normal SIP authentication sequence can include the initial `401 Unauthorized`; the second REGISTER should contain authentication credentials.

Disable logging:

```text
pjsip set logger off
```

Exit:

```text
exit
```

---

# 13. Check Endpoint 1001

Run:

```bash
sudo docker compose exec freepbx asterisk -rx "pjsip show endpoint 1001"
```

Check the output for the endpoint configuration.

Do the same for 1002:

```bash
sudo docker compose exec freepbx asterisk -rx "pjsip show endpoint 1002"
```

---

# 14. Check Contacts

Run:

```bash
sudo docker compose exec freepbx asterisk -rx "pjsip show contacts"
```

You want something representing:

```text
1001 → sip:1001@<softphone-ip>:<port>
1002 → sip:1002@<softphone-ip>:<port>
```

The exact output depends on the softphone.

---

# 15. If the Softphone Cannot Register

Check that port 5060 is listening/exposed:

```bash
sudo ss -lunp | grep 5060
```

Check Docker:

```bash
sudo docker ps
```

Check the Compose configuration:

```bash
sudo docker compose config
```

Look for the SIP port mapping.

You need the equivalent of:

```text
5060/UDP
```

between the Ubuntu host and FreePBX container.

---

# 16. Check the Ubuntu Firewall

If UFW is enabled:

```bash
sudo ufw status
```

For local testing, SIP may need:

```bash
sudo ufw allow 5060/udp
```

RTP also needs to be allowed for your configured RTP range.

For example, if your RTP range is 10000–20000:

```bash
sudo ufw allow 10000:20000/udp
```

Only expose these ports where appropriate. For a local-only development PBX, avoid opening them to the public Internet unnecessarily.

---

# 17. If the Call Connects but There Is No Audio

First check:

```bash
sudo docker compose exec freepbx asterisk -rx "rtp show settings"
```

Then check Docker port mappings.

Check your firewall:

```bash
sudo ufw status
```

Make sure your RTP range is allowed.

Also check whether both softphones are on the same LAN.

For the first test, the ideal environment is:

```text
Ubuntu
192.168.1.50
    │
    ▼
Docker / FreePBX
    │
    ├──────────────┐
    ▼              ▼
Softphone 1    Softphone 2
1001           1002
```

Do not start with phones behind different NATs.

---

# 18. Test DTMF

After audio works, test DTMF.

From 1001:

```text
1002
```

During the call, press:

```text
1
2
3
4
5
6
7
8
9
0
*
#
```

This will become important later when building:

```text
IVR
```

---

# 19. Test Hangup

From 1001:

```text
1001 → 1002
```

End the call.

Asterisk should see:

```text
BYE
200 OK
```

You can observe it using:

```bash
sudo docker compose exec freepbx asterisk -rvvv
```

and:

```text
pjsip set logger on
```

---

# 20. Test Call History

Once calls work, check FreePBX's CDR.

In the FreePBX interface:

```text
Reports
    ↓
CDR Reports
```

You should eventually see:

```text
Caller:    1001
Destination: 1002
Status:    ANSWERED
Duration:  ...
```

This is particularly important for your future PHP application.

---

# 21. Your PHP Application Comes After This

Do not initially make PHP control SIP.

Use:

```text
PHP
 │
 ├── Customers
 ├── Users
 ├── CRM
 ├── Billing
 ├── CDR
 └── Business logic
        │
        ▼
     FreePBX
        │
        ▼
     Asterisk
        │
        ▼
      PJSIP
```

Asterisk handles:

```text
SIP
RTP
Calls
Queues
IVR
Dialplan
```

FreePBX handles:

```text
PBX configuration
Extensions
Trunks
Routes
Queues
IVR configuration
```

Your PHP application handles:

```text
Application/business logic
```

---

# 22. Final Working Test

Your first milestone is complete when this works:

```text
┌─────────────────┐
│   Softphone 1   │
│                 │
│      1001       │
└────────┬────────┘
         │
         │ SIP
         │ RTP
         ▼
┌─────────────────────────┐
│       FreePBX 17        │
│                         │
│       Asterisk 22       │
│                         │
│         PJSIP           │
└────────────┬────────────┘
             │
             │ SIP
             │ RTP
             ▼
┌─────────────────┐
│   Softphone 2   │
│                 │
│      1002       │
└─────────────────┘
```

You should be able to:

```text
✓ Open FreePBX
✓ Create PJSIP extension 1001
✓ Create PJSIP extension 1002
✓ Register softphone 1001
✓ Register softphone 1002
✓ Call 1001 → 1002
✓ Hear two-way audio
✓ Send DTMF
✓ Hang up
✓ See the call in CDR
```
