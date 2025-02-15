---
title: Windows.EventLogs.ScheduledTasks
hidden: true
tags: [Client Artifact]
---

This artifact will extract Event Logs related to ScheduledTasks and provide
a nice format for simplified review.

Adversaries may abuse tasks for execution, persistence, lateral movement or
privilege escalation. This artifact collates all events from
Microsoft-Windows-TaskScheduler/Operational event log channel and scheduled
task events from the Security log if configured.

A common hunting use case may be collection all deleted scheduled tasks (EID 141),
all modified scheduled tasks (EID 140) then run frequency analysis and chase
down any abnormalities for the environment. Similarly task execution (EID 129)
and registration (EID 106) can be a good collection hunting for unusual paths.

Pivoting can be via either: TaskSchedulerEventRegex, TaskName or IOC Regex
(e.g taskname|delete|created|update)

Note: Audit Other Object Access Events is required to be implemented to record
scheduled tasks being registered, modified or disabled in the Security event
log channel.
See: Computer Configuration\Policies\Windows Settings\Security Settings\Advanced Audit Policy Configuration\Object Access


```yaml
name: Windows.EventLogs.ScheduledTasks
description: |
  This artifact will extract Event Logs related to ScheduledTasks and provide
  a nice format for simplified review.

  Adversaries may abuse tasks for execution, persistence, lateral movement or
  privilege escalation. This artifact collates all events from
  Microsoft-Windows-TaskScheduler/Operational event log channel and scheduled
  task events from the Security log if configured.

  A common hunting use case may be collection all deleted scheduled tasks (EID 141),
  all modified scheduled tasks (EID 140) then run frequency analysis and chase
  down any abnormalities for the environment. Similarly task execution (EID 129)
  and registration (EID 106) can be a good collection hunting for unusual paths.

  Pivoting can be via either: TaskSchedulerEventRegex, TaskName or IOC Regex
  (e.g taskname|delete|created|update)

  Note: Audit Other Object Access Events is required to be implemented to record
  scheduled tasks being registered, modified or disabled in the Security event
  log channel.
  See: Computer Configuration\Policies\Windows Settings\Security Settings\Advanced Audit Policy Configuration\Object Access

author: "@mgreen27 - Matt Green"

precondition: SELECT OS From info() where OS = 'windows'

reference:
  - https://attack.mitre.org/techniques/T1053/005/
  - https://mnaoumov.wordpress.com/2014/05/15/task-scheduler-event-ids/

parameters:
  - name: Security
    description: path to Security event log.
    default: '%SystemRoot%\System32\Winevt\Logs\Security.evtx'
  - name: TaskScheduler
    description: path to the TaskScheduler/Operational event log
    default: '%SystemRoot%\System32\Winevt\Logs\Microsoft-Windows-TaskScheduler%4Operational.evtx'
  - name: DateAfter
    description: "search for events after this date. YYYY-MM-DDTmm:hh:ss Z"
    type: timestamp
  - name: DateBefore
    description: "search for events before this date. YYYY-MM-DDTmm:hh:ss Z"
    type: timestamp
  - name: TaskSchedulerEventRegex
    description: Regex of TaskScheduler log event ids.
    type: regex
    default: ^(140|141)$
  - name: SecurityEventRegex
    description: regex of Security log event ids.
    type: regex
    default: '^(4698|4699|4700|4701|4702)$'
  - name: TaskNameRegex
    description: regex of target task name.
    default: .
    type: regex
  - name: TaskNameWhitelist
    description: regex of task names to exclude from results.
    default:
    type: regex
  - name: TaskActionRegex
    description: regex of target task execution process / path.
    default: .
    type: regex
  - name: TaskActionWhitelist
    description: regex of task processes to exclude from results.
    default:
    type: regex
  - name: UserNameRegex
    description: regex of target user name.
    default: .
    type: regex
  - name: IocRegex
    description: IOC regex to search for.
    default: .
    type: regex
  - name: SearchVSS
    description: "add VSS into query."
    type: bool

sources:
  - query: |
      -- firstly set timebounds for performance
      LET DateAfterTime <= if(condition=DateAfter,
        then=DateAfter, else=timestamp(epoch="1600-01-01"))
      LET DateBeforeTime <= if(condition=DateBefore,
        then=DateBefore, else=timestamp(epoch="2200-01-01"))

      -- Lookup what each task ID means (sadly dict keys are always strings).
      LET TaskIDLookup <= dict(
        `4698`="A scheduled task was created.",
        `4699`="A scheduled task was deleted.",
        `4700`="A scheduled task was enabled.",
        `4701`="A scheduled task was disabled.",
        `4702`="A scheduled task was updated.")

      -- expand provided glob into a list of paths on the file system (fs)
      LET fspaths <= SELECT FullPath
        FROM glob(globs=[expand(path=Security), expand(path=TaskScheduler)])

      -- function returning list of VSS paths corresponding to path
      LET vsspaths(path) = SELECT FullPath
        FROM Artifact.Windows.Search.VSS(SearchFilesGlob=path)

      -- function to parse task content and replace xml in EventData
      LET parse_task(data) = SELECT
                                data.SubjectUserSid as SubjectUserSid,
                                data.SubjectUserName as SubjectUserName,
                                data.SubjectDomainName as SubjectDomainName,
                                data.SubjectLogonId as SubjectLogonId,
                                data.TaskName as TaskName,
                                parse_xml(
                                    accessor='data',
                                    file=regex_replace(
                                        source= if(condition= data.TaskContentNew,
                                            then= data.TaskContentNew,
                                            else= if(condition= data.TaskContent,
                                                then= data.TaskContent)),
                                        re='<[?].+?>',
                                        replace='')).Task as TaskContent,
                                data.ClientProcessStartKey as ClientProcessStartKey,
                                data.ClientProcessId as ClientProcessId,
                                data.ParentProcessId as ParentProcessId,
                                data.RpcCallClientLocality as RpcCallClientLocality,
                               data.FQDN as FQDN
                            FROM scope()

      -- function returning query hits
      LET evtxsearch(PathList) = SELECT * FROM foreach(
            row=PathList,
            query={
                SELECT
                    timestamp(epoch=int(int=System.TimeCreated.SystemTime)) AS EventTime,
                    System.Computer as Computer,
                    System.Channel as Channel,
                    System.EventID.Value as EventID,
                    System.EventRecordID as EventRecordID,
                    if(condition=EventData.UserName,
                        then= EventData.UserName,
                        else= if(condition=EventData.UserContext,
                            then=EventData.UserContext,
                            else= if(condition=EventData.SubjectUserName,
                                then= EventData.SubjectDomainName + '\\' + EventData.SubjectUserName,
                                else= 'N/A' ))) as UserName,
                    if(condition=EventData.TaskName,
                        then= EventData.TaskName) as TaskName,
                    if(condition=EventData.ActionName,
                        then= EventData.ActionName,
                        else= if(condition=EventData.Path,
                            then= EventData.Path,
                            else= 'N/A' )) as TaskAction,
                    if(condition=EventData.TaskContent OR EventData.TaskContentNew,
                            then= parse_task(data=EventData)[0],
                            else= EventData) as EventData,
                    get(field="Message") as Message,
                    FullPath
                FROM parse_evtx(filename=FullPath)
                WHERE
                    (( Channel = 'Microsoft-Windows-TaskScheduler/Operational'
                        AND str(str=EventID) =~ TaskSchedulerEventRegex )
                    OR ( Channel = 'Security'
                        AND str(str=EventID) =~ SecurityEventRegex ))
                    AND TaskName =~ TaskNameRegex
                    AND NOT if(condition= TaskNameWhitelist,
                        then= TaskName =~ TaskNameWhitelist,
                        else= False)
                    AND TaskAction =~ TaskActionRegex
                    AND NOT if(condition= TaskActionWhitelist,
                        then= TaskName =~ TaskActionWhitelist,
                        else= False)
                    AND UserName =~ UserNameRegex
                    AND format(format='%v %v %v %v', args=[
                            EventData, UserData, Message, System]) =~ IocRegex
                    AND EventTime >= DateAfterTime AND EventTime <= DateBeforeTime
            }
          )

      -- include VSS in calculation and deduplicate with GROUP BY by file
      LET include_vss = SELECT * FROM foreach(row=fspaths,
            query={
                SELECT *
                FROM evtxsearch(PathList={
                        SELECT FullPath FROM vsspaths(path=FullPath)
                    })
                GROUP BY EventRecordID,Channel
              })

      -- exclude VSS in EvtxHunt`
      LET exclude_vss = SELECT *
        FROM evtxsearch(PathList={SELECT FullPath FROM fspaths})

      -- return rows
      SELECT
        EventTime,
        Computer,
        Channel,
        EventID,
        EventRecordID,
        UserName,
        TaskName,
        if(condition= Channel = 'Microsoft-Windows-TaskScheduler/Operational',
            then= Message,
            else=get(item=TaskIDLookup, member=str(str=EventID))) as Message,
        if(condition= str(str=EventID)=~'^(4698|4699|4700|4701|4702)$',
        then= if(condition= EventData.TaskContent.Actions,
            then= if(condition= EventData.TaskContent.Actions.Exec,
                then= if(condition= EventData.TaskContent.Actions.Exec.Arguments,
                    then= EventData.TaskContent.Actions.Exec.Command + ' ' + EventData.TaskContent.Actions.Exec.Arguments,
                    else=  EventData.TaskContent.Actions.Exec.Command),
                else= if(condition=EventData.TaskContent.Actions.ComHandler.ClassId,
                    then= 'ClassId: ' + EventData.TaskContent.Actions.ComHandler.ClassId))),
            else= TaskAction) as TaskAction,
        EventData,
        FullPath
      FROM if(condition=SearchVSS,
        then=include_vss,
        else=exclude_vss)

```
