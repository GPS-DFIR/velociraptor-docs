---
title: Admin.Client.Remove
hidden: true
tags: [Server Artifact]
---

This artifact will remove clients that have not checked in for a
while.  All data for these clients will be removed.

The artifact enumerates all the files that are removed.


```yaml
name: Admin.Client.Remove
description: |
  This artifact will remove clients that have not checked in for a
  while.  All data for these clients will be removed.

  The artifact enumerates all the files that are removed.

type: SERVER

parameters:
  - name: Age
    description: Remove clients older than this many days
    default: "7"

  - name: ReallyDoIt
    type: bool

sources:
  - query: |
      LET old_clients = SELECT os_info.fqdn AS Fqdn, client_id,
             timestamp(epoch=last_seen_at/1000000) AS LastSeen FROM clients()
      WHERE LastSeen < now() - ( atoi(string=Age) * 3600 * 24 )

      SELECT * FROM foreach(row=old_clients,
      query={
         SELECT *, Fqdn, LastSeen FROM client_delete(
             client_id=client_id, really_do_it=ReallyDoIt)
      })

```
