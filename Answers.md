# Part 2 answer:

## Answer to the Question No. 1

There are two option to resolve the error

### Option: 1 Use MySQL Binlogs (If Enabled)

Binlogs are like a "black box" of your database—they record every change. If enabled, you can "rewind" the accidental update.

Steps: <br>
<ul><li> Find the Accident
Identify the exact time the bad UPDATE ran (e.g., around 9:50 AM on April 30).
</li>
<li>Use a tool like `binlog2sql` (recommended) or `mysqlbinlog` to generate SQL that reverses the accident.</li>
<li> Example command: <br>`binlog2sql --flashback --start-datetime='2025-04-30 09:50:00' --stop-datetime='2025-04-30 09:55:00' -d your_database -t users > revert.sql`</li>

   <li>This creates a file (revert.sql) with UPDATE statements that restore the original email_verified values.</li>
   <li> Run the generated SQL in production.</li>
   <li>For example: <br>```sql UPDATE users SET email_verified = 0 WHERE user_id = 45```<br> Do the same for all unverified user</li> 
   </ul>
### Option 2: Reverse Audit Recovery (If Binlogs Are Disabled)
    • Create a Snapshot of the Last Good State:
“CREATE TEMPORARY TABLE last_good_state AS SELECT user_id, old_email_verified  
      FROM user_audit  WHERE updated_at < '2025-04-30 09:50:00';” 
    • Restore Data from the Snapshot:
“UPDATE users  JOIN last_good_state ON users.user_id = last_good_state.user_id  SET users.email_verified = last_good_state.old_email_verified;”
