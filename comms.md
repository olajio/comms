Here's an optimized version of your communication:

---

**Subject: Action Required: Default Data View Changes in Kibana Spaces**

Hello teams,

The ITSMA team will be updating the default data view across all Kibana spaces to improve cluster performance and align with Elasticsearch best practices.

**What's changing:**
- Current default: `*:filebeat-*` (or similar cross-cluster patterns)
- New default: `prod:filebeat-*` (or environment-specific equivalents)

**Why this matters:**
Our Elasticsearch vendor has advised that cross-cluster wildcard patterns (e.g., `*:filebeat-*`, `*:metricbeat-*`) can cause significant performance degradation on the cluster. Limiting data views to specific environments ensures faster searches and better cluster stability.

**Action required from your teams:**
When performing searches in Kibana, please:
1. **Avoid** selecting data views that start with `*:filebeat-*` or `*:metricbeat-*`
2. **Use** environment-specific data views instead (e.g., `prod:filebeat-*`, `qa:filebeat-*`, `dev:metricbeat-*`)
3. Update any saved searches or dashboards that use cross-cluster patterns

**Timeline:**
This change will be implemented **[INSERT DATE]**.

If you have questions or need assistance updating your searches and dashboards, please reach out to the ITSMA team.

Thank you for your cooperation in maintaining optimal cluster performance.

Best regards,  
Ola

---

**Key improvements:**
- Fixed all typos and grammar
- Added clear subject line with "Action Required"
- Structured with headers for easy scanning
- Specific before/after examples
- Clearer explanation of impact
- Actionable guidance in numbered steps
- Added timeline placeholder
- More professional tone while remaining approachable

Would you like me to adjust the tone, add more technical details, or modify any section?
