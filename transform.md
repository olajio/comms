**Perfect! This confirms "N/A" exists in your source index** - the query failed because Elasticsearch tried to parse "N/A" as a date during the query phase itself.

The failures are in these indices:
- `.ds-cdm_scuba_m365_result_trending-2025.12.08-000024`
- `.ds-cdm_scuba_m365_result_trending-2025.12.10-000025`
- `.ds-cdm_scuba_m365_result_trending-2025.12.10-000026`

## Solution: Use a script query instead

Since we can't query the date field directly for "N/A", we need to use a painless script that accesses the raw `_source`:

```json
POST cdm_scuba_m365_result_trending/_update_by_query
{
  "script": {
    "source": """
      if (ctx._source.control?.resolution_date == 'N/A' || 
          ctx._source.control?.resolution_date == '') {
        ctx._source.control.resolution_date = null;
      }
    """
  },
  "query": {
    "script": {
      "script": {
        "source": """
          def val = params._source?.control?.resolution_date;
          return (val == 'N/A' || val == '');
        """,
        "lang": "painless"
      }
    }
  }
}
```

**However**, this will be VERY slow on large indices. Better approach:

## Faster Solution: Target specific indices and use match_all

```json
POST .ds-cdm_scuba_m365_result_trending-2025.12.08-000024,
     .ds-cdm_scuba_m365_result_trending-2025.12.10-000025,
     .ds-cdm_scuba_m365_result_trending-2025.12.10-000026/_update_by_query?conflicts=proceed
{
  "script": {
    "source": """
      if (ctx._source.control?.resolution_date == 'N/A' || 
          ctx._source.control?.resolution_date == '') {
        ctx._source.control.resolution_date = null;
      }
    """
  },
  "query": {
    "match_all": {}
  }
}
```

This will:
1. Scan all documents in those 3 problem indices
2. Check each document's `_source` 
3. Update only those with "N/A" or empty strings
4. Leave valid dates unchanged

## After the update completes:

1. **Stop the transform:**
   ```json
   POST _transform/cdm_scuba_m365_result/_stop
   ```

2. **Reset the transform:**
   ```json
   POST _transform/cdm_scuba_m365_result/_reset
   ```

3. **Start the transform:**
   ```json
   POST _transform/cdm_scuba_m365_result/_start
   ```

## Prevent future occurrences:

Check what's writing "N/A" to `cdm_scuba_m365_result_trending`. Look for:
- Logstash configs
- Beats configurations
- Application code
- Another ingest pipeline

The root cause is something is indexing the literal string "N/A" into a date field instead of using `null`.

Try the targeted update and let me know the result!





**Yes, absolutely!** The ingest pipeline fix is critical for a permanent solution. Here's why:

## Two-Part Solution Needed:

### 1. **update_by_query** (One-time fix)
- Cleans up **existing** "N/A" values already in the index
- But does **nothing** to prevent new "N/A" values from arriving

### 2. **Ingest Pipeline** (Permanent protection)
- Catches **future** "N/A" values before they break the transform
- Acts as a safety net for ongoing data ingestion

## Without the ingest pipeline:
- New documents with "N/A" will continue flowing into `cdm_scuba_m365_result_trending`
- Your transform will fail again within hours/days
- You'll be stuck in a loop of manually cleaning data

## Add this to your pipeline:

Update `cdm_scuba_m365_result_final` to add this as the **FIRST** processor:

```json
PUT _ingest/pipeline/cdm_scuba_m365_result_final
{
  "description": "Ingest pipeline for the latest CDM SCUBA M365 result data.",
  "processors": [
    {
      "script": {
        "description": "Clean invalid date values before processing",
        "source": """
          // Handle N/A and empty strings in date fields
          if (ctx.control?.resolution_date == 'N/A' || 
              ctx.control?.resolution_date == '') {
            ctx.control.remove('resolution_date');
          }
        """
      }
    },
    {
      "script": {
        "source": """
          String errors="";
          // ... your existing validation script ...
        """
      }
    }
    // ... rest of your existing processors ...
  ]
}
```

## The Real Root Cause:

You should also investigate **what's writing "N/A"** to `cdm_scuba_m365_result_trending`:
- Check the ingest pipeline for that index
- Check Logstash/Beats configs
- Check application code

Fixing the source would be ideal, but the ingest pipeline ensures your transform stays resilient even if bad data slips through.

**Bottom line: Do both the update_by_query AND the ingest pipeline fix for complete protection.**







***
**The fact that the update_by_query FAILED proves the "N/A" values ARE there** - Elasticsearch couldn't even run the query because it tried to parse "N/A" as a date!

Here's the fix: **Use `match_all` and let the script do the filtering**:

## Solution: Update ALL documents, script filters for "N/A"

```json
POST .ds-cdm_scuba_m365_result_trending-2025.12.08-000024,
     .ds-cdm_scuba_m365_result_trending-2025.12.10-000025,
     .ds-cdm_scuba_m365_result_trending-2025.12.10-000026/_update_by_query?conflicts=proceed&wait_for_completion=false
{
  "script": {
    "source": """
      if (ctx._source.control?.resolution_date != null) {
        String val = ctx._source.control.resolution_date.toString();
        if (val == 'N/A' || val == '' || val.trim() == '') {
          ctx._source.control.resolution_date = null;
        }
      }
    """,
    "lang": "painless"
  },
  "query": {
    "match_all": {}
  }
}
```

**What this does:**
- Scans **every** document in those 3 indices
- Checks each document's `_source` (raw data, no parsing)
- Only updates documents where `control.resolution_date` is "N/A" or empty
- Leaves valid dates completely untouched
- Returns immediately with a task ID (runs in background)

## Check progress:

```json
GET _tasks?detailed=true&actions=*byquery
```

## Alternative: If the above is too slow, target by date range

If you know roughly when the bad data arrived:

```json
POST .ds-cdm_scuba_m365_result_trending-2025.12.08-000024,
     .ds-cdm_scuba_m365_result_trending-2025.12.10-000025,
     .ds-cdm_scuba_m365_result_trending-2025.12.10-000026/_update_by_query?conflicts=proceed
{
  "script": {
    "source": """
      if (ctx._source.control?.resolution_date != null) {
        String val = ctx._source.control.resolution_date.toString();
        if (val == 'N/A' || val == '' || val.trim() == '') {
          ctx._source.control.resolution_date = null;
        }
      }
    """
  },
  "query": {
    "range": {
      "@timestamp": {
        "gte": "2025-12-08",
        "lte": "2025-12-11"
      }
    }
  }
}
```

## After it completes:

```json
POST _transform/cdm_scuba_m365_result/_stop
POST _transform/cdm_scuba_m365_result/_reset
POST _transform/cdm_scuba_m365_result/_start
```

**Key insight:** We can't FIND the documents with normal queries, but `match_all` + script filtering will UPDATE them successfully. Try this and let me know the result!


## NEW

Looking at the transform stats, I can see it's already processed 133,429 documents and indexed 30,000 successfully before hitting the "N/A" issue. The transform is stuck at a specific position:

```
"indexer_position": {
  "control.id": "MS.TEAMS.7.1v1",
  "scan.date": 1757446069724,
  "tenant.id": "0d69cd8a-9e12-4d21-95f0-b4406e06710a"
}
```

## The Problem

The document ID `TQAyX367H8rLew5F05CihqdOeAAAAAAA` is **generated by the transform itself** (it's not the source document's `_id`). This is why you can't find it in the source index!

## Find the Actual Source Document

Use the transform's unique key to find the problematic document:

```json
GET cdm_scuba_m365_result_trending/_search
{
  "_source": true,
  "query": {
    "bool": {
      "must": [
        { "term": { "control.id.keyword": "MS.TEAMS.7.1v1" } },
        { "term": { "tenant.id.keyword": "0d69cd8a-9e12-4d21-95f0-b4406e06710a" } },
        { "term": { "scan.date": 1757446069724 } }
      ]
    }
  }
}
```

This should return the exact document causing the issue. Once you find it, examine the `control.resolution_date` field value.

## Fix That Specific Document

Once you find the document's real `_id`, update it:

```json
POST cdm_scuba_m365_result_trending/_update/<ACTUAL_DOC_ID>
{
  "script": {
    "source": """
      if (ctx._source.control?.resolution_date == 'N/A' || 
          ctx._source.control?.resolution_date == '') {
        ctx._source.control.resolution_date = null;
      }
    """
  }
}
```

## Or Cast a Wider Net

If the above query doesn't find it (maybe `scan.date` is stored differently), try:

```json
POST cdm_scuba_m365_result_trending/_update_by_query?conflicts=proceed
{
  "script": {
    "source": """
      if (ctx._source.control?.resolution_date != null) {
        String val = ctx._source.control.resolution_date.toString();
        if (val == 'N/A' || val == '' || val.trim() == '') {
          ctx._source.control.resolution_date = null;
        }
      }
    """
  },
  "query": {
    "bool": {
      "must": [
        { "term": { "control.id.keyword": "MS.TEAMS.7.1v1" } },
        { "term": { "tenant.id.keyword": "0d69cd8a-9e12-4d21-95f0-b4406e06710a" } }
      ]
    }
  }
}
```

## After Fixing

```json
POST _transform/cdm_scuba_m365_result/_stop
POST _transform/cdm_scuba_m365_result/_reset
POST _transform/cdm_scuba_m365_result/_start
```

**Try the search query first to see the actual document and its `control.resolution_date` value.** Let me know what you find!
