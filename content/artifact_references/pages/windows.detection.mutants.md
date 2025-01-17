---
title: Windows.Detection.Mutants
hidden: true
tags: [Client Artifact]
---

Enumerate the mutants from selected processes.

Mutants are often used by malware to prevent re-infection.


```yaml
name: Windows.Detection.Mutants
description: |
  Enumerate the mutants from selected processes.

  Mutants are often used by malware to prevent re-infection.

parameters:
  - name: processRegex
    description: A regex applied to process names.
    default: .
    type: regex
  - name: MutantNameRegex
    default: .+
    type: regex
  - name: MutantWhitelistRegex
    default:
    type: regex

sources:
  - name: Handles
    description: Open handles to mutants. This shows processes owning a handle open to the mutant.
    query: |
        LET processes = SELECT Pid AS ProcPid, Name AS ProcName, Exe
        FROM pslist()
        WHERE ProcName =~ processRegex AND ProcPid > 0

        SELECT * FROM foreach(
          row=processes,
          query={
            SELECT ProcPid, ProcName, Exe, Type, Name, Handle
            FROM handles(pid=ProcPid, types="Mutant")
          })
        WHERE Name =~ MutantNameRegex
            AND if(condition= MutantWhitelistRegex,
                then= NOT Name =~ MutantWhitelistRegex,
                else= True )

  - name: ObjectTree
    description: Reveals all Mutant objects in the Windows Object Manager namespace.
    query: |
        SELECT Name, Type FROM winobj()
        WHERE Type = 'Mutant' AND Name =~ MutantNameRegex
            AND if(condition= MutantWhitelistRegex,
                then= NOT Name =~ MutantWhitelistRegex,
                else= True )

```
