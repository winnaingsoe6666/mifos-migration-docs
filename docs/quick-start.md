# Quick Start Guide

## မြန်မြန် စတင်နည်း

### Prerequisites
- Python 3.x installed
- MySQL client installed
- DBeaver installed
- Access to AWS Lightsail

### Quick Steps

1. **Split the 4GB file**
```bash
   python split_sql_dump.py
```

2. **Fix character encoding**
```bash
   # Use PowerShell script or manual find/replace
```

3. **Compare schemas**
   - Open DBeaver
   - Connect to databases
   - Run comparison

4. **Run ETL**
```bash
   mysql -u user -p < migrate_data.sql
```

5. **Validate**
```sql
   -- Run all validation queries
```

### Emergency Contacts

- Technical Lead: [Phone/Email]
- Client Contact: [Phone/Email]
- AWS Support: [Account details]

### Quick Commands
```sql
-- Check row counts
SELECT COUNT(*) FROM table_name;

-- Check character encoding
SHOW VARIABLES LIKE 'character%';

-- Monitor processes
SHOW PROCESSLIST;
```

## Next Steps

Read the [Full Migration Plan](migration-plan.md) for detailed instructions.