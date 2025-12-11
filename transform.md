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
