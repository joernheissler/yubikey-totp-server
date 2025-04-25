###################
YubiKey TOTP server
###################

Purpose of this project is to provide restricted access to the `OATH feature of a YubiKey
<https://docs.yubico.com/yesdk/users-manual/application-oath/oath-overview.html>`_ via
Secure Shell.

Clients connect through SSH and authenticate with their public key. The program responds with
the current TOTP code.

Multiple clients and YubiKeys are supported, along with access control lists (who may access
which code).

Dependencies
============
* python3
* python3-platformdirs
* python3-yaml
* python3-ykman
* yubikey-manager

One time setup
==============
The OATH app needs to be setup once::

    $ ykman list
    YubiKey 5 Nano (5.2.7) [OTP+FIDO+CCID] Serial: 12345678

    $ ykman -d 12345678 info | grep ^OATH
    OATH    Disabled

    # If the App is "Disabled", enable it:

    $ ykman -d 12345678 config usb --enable oath
    Enable OATH.
    Configure USB? [y/N]: y

    # Reset the application and all stored keys, and create a new "Name" (8 byte random) for the app.

    $ ykman -d 12345678 oath reset
    WARNING! This will delete all stored OATH accounts and restore factory settings. Proceed? [y/N]: y
    Resetting OATH data...
    Success! All OATH accounts have been deleted from the YubiKey.

    $ ykman -d 12345678 oath access change
    Enter the new password: start123    # hidden
    Repeat for confirmation: start123   # hidden
    Password updated.

    # Optionally, you can store the (derived) access code on disk:
    # ~/.ykman/oath.json or ~/.local/share/ykman/oath_keys.json depending on ykman version.

    $ ykman -d 12345678 oath access remember
    Enter the password: start123    # hidden
    Password remembered.

Adding secret
=============
Add a new secrets::

    # Use --help to get additional options. E.g. -t to require touch.

    $ ykman -d 12345678 oath accounts add -o TOTP -d 6 -a SHA1 -P 30 name_of_secret
    Enter a secret key (base32): JBSWY3DPFQQFO33SNRSCCIB2FE

To test the functionality, execute::

    $ ykman -d 12345678 oath accounts code -s name_of_secret
    079361

Configuration
=============

User
----
Create a dedicated user on your system to run this program. Access to the YubiKey is
required, e.g. using a udev rule or through pcscd. This is out of scope for this manual.

If ``ykman`` works, ``yubikey-totp-server`` should as well.

OpenSSH
-------
Add this to your opensshd configuration::
    
    ExposeAuthInfo yes

Then restart opensshd.

authorized_keys
---------------
Into ``~/.ssh/authorized_keys`` add for each permitted public key::

    command="/path/to/yubikey-totp-server" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFVXtAoiw1Q09Vzv+LKSvxoKMxLwOED30u3djfBUg8at


Configuration file path: ``~/.config/yubikey-totp-server/config.yaml``::

    ---
    # Mapping from yubikey serials to oath access codes:
    devices:
      12345678: "start123"

    # Mapping from secret-aliases to device-serial + secret-name
    secrets:
      some_alias:
        dev: 12345678
        name: name_of_secret

    # Mapping from SSH public keys to allowed secret-aliases.
    acl:
      AAAAC3NzaC1lZDI1NTE5AAAAIFVXtAoiw1Q09Vzv+LKSvxoKMxLwOED30u3djfBUg8at:
        - some_alias


Client usage
============
On the client, run::

    $ ssh totp@your-server some_alias
    079361
