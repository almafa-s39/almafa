# IPsec crypto settings

## For swanctl.conf

```
connections {
    s2s {
        .....
        version = 2
        proposals = aes256gcm16-prfsha512-ecp521
        ...
        children {
            net {
                .....
                esp_proposals = aes256gcm16-prfsha512-ecp521
            }
        }
    }
}
```

## For ipsec.conf

```
conn xyz
    ....
    keyexchange=ikev2
    ike=aes256gcm16-prfsha512-ecp521
    esp=aes256gcm16-prfsha512-ecp521
    ...

```
