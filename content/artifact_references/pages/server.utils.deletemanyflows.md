---
title: Server.Utils.DeleteManyFlows
hidden: true
tags: [Server Artifact]
---

Sometimes the Velociraptor server accumulates a lot of data that is
no longer needed.

This artifact will enumerate all flows from all clients and matches
them against some criteria. Flows that match are then removed.

**NOTE** This artifact will destroy all data irrevocably. Take
  care! You should always do a dry run first to see which flows
  will match before using the ReallyDoIt option.


```yaml
name: Server.Utils.DeleteManyFlows
description: |
   Sometimes the Velociraptor server accumulates a lot of data that is
   no longer needed.

   This artifact will enumerate all flows from all clients and matches
   them against some criteria. Flows that match are then removed.

   **NOTE** This artifact will destroy all data irrevocably. Take
     care! You should always do a dry run first to see which flows
     will match before using the ReallyDoIt option.

type: SERVER

parameters:
   - name: ArtifactRegex
     default: Generic.Client.Info
     type: regex
   - name: HostnameRegex
     description: If specified only target these hosts
     type: regex
   - name: DateBefore
     default: "2022-01-01"
     type: timestamp
   - name: CreatorRegex
     default: "H\\..+"
     type: regex
     description: |
       Match flows created by this user (e.g. hunts all start with "H.")
   - name: ReallyDoIt
     type: bool
     description: Does not delete until you press the ReallyDoIt button!

sources:
  - query: |
        LET hits = SELECT * FROM foreach(row={
            SELECT client_id,
                   os_info.hostname AS hostname
            FROM clients()
            WHERE hostname =~ HostnameRegex
        },
        query={
          SELECT client_id, hostname,
                 session_id, request.creator AS creator,
                 request.artifacts as artifacts,
                 timestamp(epoch=create_time) AS created
          FROM flows(client_id=client_id)
          WHERE creator =~ CreatorRegex
             AND artifacts =~ ArtifactRegex
             AND created < DateBefore
        }, workers=10)

        SELECT * FROM if(condition=ReallyDoIt,
        then={
            SELECT * FROM foreach(row=hits,
            query={
                SELECT client_id, hostname, creator,
                       session_id, artifacts, created, Type, deleted
                FROM Artifact.Server.Utils.DeleteFlow(
                   ClientId=client_id,
                   FlowId=session_id,
                   ReallyDoIt=ReallyDoIt)
                WHERE log(message=format(format="Deleting flow %v from %v",
                   args=[session_id, hostname]))
            }, workers=10)
        }, else={
            SELECT * FROM hits
        })

```
