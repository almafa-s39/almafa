<!-- 
---
title: "Syslog-NG"
author: "Gergő Téringer"
---
 -->
# Syslog-NG

## Basics

>[!NOTE]
> You have to install `syslog-ng` package to use the daemon.
> There are some predefined **sources**, **filters** and **destinations** in `/etc/syslog-ng/syslog-ng.conf` file.
> For modularity and seperation, if you create a new config, then please place it under `/etc/syslog-ng/conf.d` directory and name it `*.conf`, because the main configuartion file will include these config files.

### Source

>[!NOTE]
> The sources of logging. The default source where you get all local system logs is `s_src`.
> Later we will define sources in these to achieve log collection from syslog clients.

#### Syntax

```bash
source s_name {
    system();
};
```

### Filter

> [!NOTE]
> A filter you can define when you want to place different daemon logs to different files.

#### Syntax

```bash
filter f_name {
    level(info) and /\ or /\ not facility() and /\ or /\ not program();
};
```

### Destination

> [!NOTE]
> The destination where you place, or where you send your logs. On servers you will define files, and on client you will define mainly transport options.

>[!TIP]
> If you want to place every log into a tty line, you can define a destination to `/dev/ttyX` where **X** means the tty's number.


#### Syntax

```bash
destination d_name {
    file("/dev/tty5");
};
```

### Log

> [!NOTE]
> If you want a new log into your server, than you have to use the `log` keyword. In a `log` field you will use one or more (or zero) predefined **source**, **filter** and **destination**

#### Syntax

```bash
log{
    source(s_name);
  filter(f_name); # Optional
  destination(d_name);
};
```

## Sending logs over network

### IETF

> [!NOTE]
> One protocol from the two with you can send and receive syslogs. You have to use the `syslog` keyword to use this. In a `syslog` field you will use one or more (or zero) predefined **source**, **filter** and **destination**. You have to define this in a source or a destination.

#### Syntax

```bash
syslog{
    ip-protocol(4) # If you define 6 it wil listen on IPv6 and IPv4 too.
    port(6514) # Number between 1-65536
    transport("tls") # udp,tcp,tls
    tls (
        cert-file("/ca/SRV.pem")
      key-file("/ca/SRV.key")
      ca-file("/ca/CA.crt")
      ca-dir("/ca/")
      # peer-verify(optional-untrusted); # You can define this, there will be a table under this what will provide which option do what.
};
```

### IETF

> [!NOTE]
> One protocol from the two with you can send and receive syslogs. You have to use the `syslog` keyword to use this. In a `syslog` field you will use one or more (or zero) predefined **source**, **filter** and **destination**

#### Syntax

```bash
syslog{
    source(s_name);
  filter(f_name); # Optional
  destination(d_name);
};
```

### BSD

> [!NOTE]
> One protocol from the two with you can send and receive syslogs. You have to use the `network` keyword to use this. In a `network` field you will use one or more (or zero) predefined **source**, **filter** and **destination**. You have to define this in a source or a destination.

#### Syntax

```bash
network{
    ip-protocol(4) # If you define 6 it wil listen on IPv6 and IPv4 too.
    port(6514) # Number between 1-65536
    transport("tls") # udp,tcp,tls
    tls (
        cert-file("/ca/SRV.pem")
      key-file("/ca/SRV.key")
      ca-file("/ca/CA.crt")
      ca-dir("/ca/")
      # peer-verify(optional-untrusted); # You can define this, there will be a table under this what will provide which option do what.
};
```

### Peer-verify()

> [!NOTE]
> The deafult value is **required-trusted**.

| Option             | No cert             | Invalid cert        | Valid cert     |
| :----------------- | :-----------------: | :-----------:       | :---------:    |
| optional-untrusted | TLS-encryption      | TLS-encryption      | TLS-encryption |
| optional-trusted   | TLS-encryption      | rejected connection | TLS-encryption |
| require-untrusted  | rejected connection | TLS-encryption      | TLS-encryption |
| require-trusted    | rejected connection | rejected connection | TLS-encryption |

## Transport over TLS

> [!NOTE]
> You have to generate [x509 certificates](/cert/openssl). I recommend to create chains from certificates.
> After the creation make the devices trust the CA certificate, and start to configure your devices.
> [HERE](https://syslog-ng.github.io/admin-guide/100_TLS-encrypted_message_transfer/004_TLS_options) you can check every TLS option.

### Server side configuration

```bash
source s_dhcp {
    syslog(
    ip-protocol(4)
    port(6514)
    transport("tls")
    tls (
        cert-file("/ca/SRV.pem")
      key-file("/ca/SRV.key")
      ca-file("/ca/CA.crt")
      ca-dir("/ca/")
    )
  );
};

destination d_dhcp {
    file("/log/dhcp.log");
};

log{
    source(s_dhcp);
  destination(d_dhcp);
};
```

### Client side configuration

```bash
destination d_dhcp{
    syslog(
      "SRV.lego.dk"
        port(6514)
      transport("tls")
      tls(
          cert-file("/ca/CLT.pem")
        key-file("/ca/CLT.key")
        ca-file("/ca/CA.crt")
      )
  );
};

filter f_dhcp{
    program("dhcpd") or program("dhclient");
};

log {
    source(s_src);
  filter(f_dhcp);
  destination(d_dhcp);
  
  # ( or if you want to send everything except your filter)
  # filter { 
  #      not filter(f_dhcp)
  # };
};
```

<!-- Created by: Gergő Téringer, 2026 -->