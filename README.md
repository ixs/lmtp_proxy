# lmtp_proxy
A simple LMTP delivery proxy that is able to deliver to different backends depending on the rcpt to user


## Function
LMTP is often used for for communication between message transfer agent and message delivery agent/local delivery agent. https://en.wikipedia.org/wiki/Local_Mail_Transfer_Protoc$

When doing maintenance or upgrades on an SMTP system it might be useful to be able to redirect delivery temporarily to a different storage backend.

This LMTP proxy was written to do a no-downtime migration of users from a cyrus based IMAP server to a newer version of the IMAP server.
Perdition is used to redirect the end users from the old to the new imap service while this lmtp proxy is used to redirect the incoming mail
from postfix to the right backend store.


## Commandline parameters
```
usage: lmtp-proxy [-h] [-c [CONFIG]] [-D]

An LMTP Proxy server. Accepts mail via LMTP on an incoming socket and can
forward to different lmtp sockets depending on the destination address.

optional arguments:
  -h, --help            show this help message and exit
  -c [CONFIG], --config [CONFIG]
                        Configuration file location
  -D, --daemonize       Deamonize program. If unspecified, run in foreground
```


## Configuration
A configuration file is necessary to run the proxy. As a format for the config yaml was chosen as it can be easily updated
via scripts.

* config: Basic configuration items for lmtp-proxy.
  * socket: Either a full path to the incoming unix domain socket or a list of two items, host and port.
  * owner: The owner of the unix domain socket. Ignored when not using a unix domain socket.
  * group: The group owner of the unix domain socket. Ignored when not using a unix domain socket.
  * permissions: Permissions of the unix domain socket file. Ignored when not using a unix domain socket. Needs to be in octal, e.g. 0660 instead of just 660.
  * ignoreDomain: Boolean to specify whether the full destination address (user@example.com) is used for the user table lookup or just the localpart (user).
  * fallback_backend: The backend to use in case the user could not be found in the lookup table.
  * pid: PID file location when using --daemonize, defaults to /var/run/lmtp-proxy.pid when unspecified.
* backends: Dictionary of backend names each containing the following configuration items:
  * socket: Full path of the local unix domain socket to reach the backend LMTP server.
  * host: Hostname of backend server. Ignored if using a local unix domain socket.
  * port: Port of the backend server. Ignored if using a local unix domain socket.
  * user: Username for authentication if needed for LMTP delivery.
  * password: Password for authentication if needed for LMTP delivery.
* users: Mapping table of users. username: backendname.
  `reject` is a special backend, which will send a 4xx status code back to the LMTP client indicating a temporary error and instruct the service to try again.


## Signals

* SIGUSR1: Reload the configuration. This will effectively update the list of backends and the user mapping table
  This allows for changing the user mapping table without having to restart the lmtp daemon.


## Minmal downtime migration of users

### System setup
```
                                - imap -> cyrus-old <- lmtp -
                              /                               \
enduser - imap -> perdition -                                   - lmtp-proxy <- mta/postfix
                              \                               /
                                - imap -> cyrus-new <- lmtp -
```

### Steps
1. Migrate imap store from old to new server, e.g. using imapsync
2. Prevent the user from accessing the imap store either by temporarily modifying the password
   or by changing the perdition redirection
3. Prevent mail from being delivered to the imap store using the reject backend of the lmtp-proxy
4. Final imapsync, both stores are now completely in sync
5. Have mail delivery go to the new imap store using lmtp-proxy
6. Configure perdition to redirect the user to the new imap store
7. If applicable, reset user password to allow login

If done scripted, downtime for the user itself is only the final imapsync which usually is less than 10 seconds.
The full system does not experience any downtime at all.
