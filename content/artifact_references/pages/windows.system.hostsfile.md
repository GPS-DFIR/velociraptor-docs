---
title: Windows.System.HostsFile
hidden: true
tags: [Client Artifact]
---

Parses the Windows Hostsfile.

Regex searching for Hostname and resolutin is enabled over output.
NOTE: For Hostname search is on the hostfile line and regex ^ or $
is not reccomended.


```yaml
name: Windows.System.HostsFile
author: Matt Green - @mgreen27
description: |
   Parses the Windows Hostsfile.

   Regex searching for Hostname and resolutin is enabled over output.
   NOTE: For Hostname search is on the hostfile line and regex ^ or $
   is not reccomended.

type: CLIENT

parameters:
  - name: HostsFile
    default: C:\Windows\System32\drivers\etc\hosts
  - name: HostnameRegex
    description: "Hostname target Regex in Hostsfile"
    default: .
    type: regex

  - name: ResolutionRegex
    description: "Resolution target Regex in Hostsfile"
    default: .
    type: regex

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
      -- Parse hosts file
      Let lines = SELECT split(string=Data,sep='\\r?\\n|\\r') as List
        FROM read_file(filenames=HostsFile)

      -- extract into fields
      LET results = SELECT * FROM foreach(row=lines,
                query={
                    SELECT parse_string_with_regex(
                        string=_value,
                        regex=[
                            "^\\s*(?P<Resolution>[^\\s]+)\\s+" +
                            "(?P<Hostname>[^\\#]+)\\s*" +
                            "#*\\s*(?P<Comment>.*)$"
                        ]) as Record
                    FROM foreach(row=List)
                    WHERE _value
                        AND NOT _value =~ '^\\s*#'
                        AND _value =~ HostnameRegex
                        AND _value =~ ResolutionRegex
                })

      -- clean up hostname output
      LET hostlist(string)=
            if(condition= len(list=split(string=regex_replace(source=string,
                    re='\\s+$', replace=''), sep='\\s+')) = 1,
                then= regex_replace(source=string,re='\\s+$', replace=''),
                else= split(string=regex_replace(source=string,re='\\s+$',
                  replace=''), sep='\\s+'))

      -- output rows
      SELECT
        Record.Resolution AS Resolution,
        hostlist(string=Record.Hostname) AS Hostname,
        Record.Comment AS Comment
      FROM results

```
