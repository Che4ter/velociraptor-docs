---
title: Windows.Search.VSS
hidden: true
tags: [Client Artifact]
---

This artifact will find all relevant files in the VSS. Typically used to
out deduplicated paths for processing by other artifacts.

Input either search Glob or FullPath.
Output is standard Glob results with additional fields:
SHA1 hash for deduplication,
Type for prioritisation, and
Deduped to indicate if FullPath has been deduped with another row.


```yaml
name: Windows.Search.VSS
description: |
  This artifact will find all relevant files in the VSS. Typically used to
  out deduplicated paths for processing by other artifacts.

  Input either search Glob or FullPath.
  Output is standard Glob results with additional fields:
  SHA1 hash for deduplication,
  Type for prioritisation, and
  Deduped to indicate if FullPath has been deduped with another row.

author: Matt Green - @mgreen27

precondition: SELECT * FROM info() where OS = 'windows'

parameters:
  - name: SearchFilesGlob
    default: C:\Windows\System32\winevt\Logs\Security.evtx
    description: Use a glob to define the files that will be searched.

sources:
  - query: |
      -- Given a path in either device notation or drive notation,
      -- break it into a drive and path
      LET extract_path(FullPath) = parse_string_with_regex(string=FullPath,
         regex="^(?P<Device>\\\\\\\\.\\\\GLOBALROOT\\\\Device\\\\HarddiskVolumeShadowCopy[^\\\\]+\\\\|\\\\\\\\.\\\\.:\\\\|.:\\\\)(?P<Path>.+)$")

      -- Remove possible drive or device from the glob path - we will
      -- apply the glob to all devices.
      LET StrippedGlob <= extract_path(FullPath=SearchFilesGlob).Path

      -- Get all logical disks and VSS
      LET roots = SELECT OSPath AS Root
        FROM glob(globs='*', accessor='ntfs')
        ORDER BY Root  DESC

      -- Glob for all files in SearchGlob and calculate their hash.
      LET globvss(SearchGlob, Root) = SELECT *,
            extract_path(FullPath=OSPath).Path AS Path,
            Root.String AS Source,
            hash(path=OSPath, accessor='ntfs').SHA1 as SHA1
         FROM glob(globs=SearchGlob, accessor='ntfs', root=Root)
         WHERE NOT IsDir

      -- For each full glob (including VSS device) extract all files
      -- and hashes.
      LET results = SELECT * FROM foreach(row=roots,
      query={
        -- Prepend VSS with _ to make them sort last.
        SELECT *,
           if(condition=Source =~ '^HarddiskVolumeShadowCopy',
              then='_' + Source,
              else=Source) AS Source
        FROM globvss(SearchGlob=StrippedGlob, Root=Root)
      })

      -- We want to see natural drives after VSS because group by
      -- shows the last in the group. VSS Sources look like
      -- HarddiskVolumeShadowCopy1 and disk sources look like C:
      ORDER BY Source DESC

      -- Dedup and show results
      SELECT *, count() > 1 AS Deduped, SHA1 + Path AS Key
      FROM results
      GROUP BY Key

```
