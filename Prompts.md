Run a comprehensive analysis of clickhouse server performance for 2026-01-08 before 10:00. The server is accessible over ai-demo connector.  Look for errors, explore select and insert query performance, ingestion behaviour, problems with merges and mutations.  If errors are found, go deeper and find a root cause. search github for relevant issues. Write all findings and recommendations into md file. don't read/use old reports.
---
Please take this URL: https://clickhouse.com/blog/updates-in-clickhouse-2-sql-style-updates and produce an operator-facing HOWTO from it without pasting the full article into chat. Do it as a background pipeline: fetch the page, extract plain text, chunk it (chunk-chars=9000), run a goal-driven summarization pass, then reduce to a single howto (final-max-words=650). Goal: prerequisites/compatibility, enabling flags/settings, safe rollout checklist, 2–4 minimal example commands/SQL, verification steps/queries, and gotchas/limits. Exclude design rationale, benchmarks, images, proofs, and marketing fluff. Save to summaries/path-updates-howto.md and paste the final howto content into this chat.
---
Why did the bug touch only the dim_practitioners table? 
Read docs -  https://clickhouse.com/docs/guides/developer/on-the-fly-mutations and the file summaries/path-updates-howto.md to understand better patch parts usage
After that, go deeper into the LOGICAL_ERROR problem.
Verify schema, queries, and settings for the correct usage of the new patch UPDATE/DELETE feature.  
Compare update/delete queries to different tables and try to find the difference. show examples for the subsequent discussion.
Create a separate report related LOGICAL_ERROR problem.



