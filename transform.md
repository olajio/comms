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
