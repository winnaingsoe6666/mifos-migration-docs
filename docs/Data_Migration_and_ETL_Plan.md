# Mifos Data Migration & ETL Plan (Myanmar Translation)

## Project Overview

Hey team! ဒါက client ရဲ့ Mifos အဟောင်းကနေ ငါတို့ရဲ့ AWS Lightsail မှာ run နေတဲ့ Mifos အသစ်ကို data migrate လုပ်ဖို့ complete plan ပါ။

### လက်ရှိအခြေအနေ
- **Client**: သူတို့ဆီမှာ လက်ရှိ Mifos project အဟောင်း run နေတယ်
- **ငါတို့ Status**: Development ပြီးပြီ၊ အခု bug fix phase မှာ ရှိတယ်
- **Migration လိုအပ်ချက်**: Client ရဲ့ 4GB SQL backup ကို ငါတို့ system အသစ်ကို migrate လုပ်ရမယ်
- **Access မရှိ**: Client ရဲ့ project အဟောင်းကို ကြည့်ခွင့်မရှိဘူး၊ SQL backup file ပဲ ရှိတယ်
- **Infrastructure**: AWS Lightsail သုံးထားတယ် - Tomcat (backend WAR), Angular Docker (frontend), MySQL database
- **File အခြေအနေ**: 4GB SQL file ကြီး - DDL (schema) နဲ့ DML (data) နှစ်ခုလုံး ပါတယ်

---

## ငါတို့ကြုံတဲ့ အဓိက စိန်ခေါ်မှုများ

Plan ဒါ မစခင်၊ အဓိက စိန်ခေါ်မှုတွေ ၃ ခုကို နားလည်ထားရမယ်:

### 1. Schema ကွာခြားမှု
- Client ရဲ့ Mifos version အဟောင်းနဲ့ ငါတို့ version အသစ် မတူနိုင်ဘူး
- Database schemas တွေ (table name, column name, data type) ကွာခြားနိုင်တယ်
- Client ရဲ့ project အဟောင်းကို ကြည့်ခွင့်မရှိတော့ SQL file ဒါမှနေပဲ ရှာရမယ်

### 2. File Size ကြီးတာ (4GB)
- 4GB SQL text file ကို တိုက်ရိုက် import လုပ်တော့ အချိန်အကြာကြီး ကြာတယ်
- Lightsail ရဲ့ RAM/CPU က မလောက်လို့ fail ဖြစ်ချေ့်းနိုင်တယ်
- ဒါကြောင့် smart ဖြတ်ပြီး process လုပ်ရမယ်

### 3. DDL + Data ရောပြီး ထည်းထားတဲ့
- SQL file မှာ CREATE TABLE statements တွေနဲ့ INSERT statements တွေ ရောပြီး ထည်းထားတယ်
- အဟောင်း DDL ကို အသစ် database မှာ တိုက်ရိုက် run လို့ မရဘူး - conflict ဖြစ်မယ်
- ငါတို့ Mifos အသစ်က သူ့ schema ကို generate လုပ်ပြီးသားပဲ

### 4. Character Encoding (အရေးကြီးဆုံး!)
- **Client ရဲ့ file**: `utf8` သုံးထားတယ် (တကယ်တော့ `utf8mb3` - 3-byte)
- **ငါတို့ system အသစ်**: `utf8mb4` (4-byte) သုံးရမယ် future-proof ဖြစ်ဖို့
- မြန်မာစာက `utf8` မှာ အဆင်ပြေတယ်၊ ဒါပေမဲ့ emojis တွေ special characters တွေက ပျက်ချွေးမယ်
- အရာအားလုံးကို `utf8mb4` ကို convert လုပ်ရမယ်

---

## Phase 1: ကြိုတင်ပြင်ဆင်ခြင်းနှင့် ခွဲခြမ်းစိတ်ဖြာခြင်း

ဒါက အရေးကြီးဆုံး phase ပါ။ ဒီမှာ အချိန် ယူပြီး သေချာစွာ လုပ်ပါ!

### Step 1.1: Client ဆီမှ အချက်အလက် တောင်းခံတဲ့ (မဖြစ်မနေ လုပ်ရမယ်)

Client ကို ဆက်သွယ်ပြီး မေးပါ:
- **Mifos version number** ဘာသုံးနေလဲ (schema ကွာခြားမှုကို သိဖို့ အရေးကြီးတယ်)
- **Database character set** ဘာလဲ (ငါတို့ `utf8` ဆိုတာ ထေး့ပြီးပြီ၊ ဒါပေမဲ့ confirm လုပ်ပါ)
- **Custom modifications** ဘာတွေ လုပ်ထားလဲ သူတို့ Mifos installation မှာ

### Step 1.2: 4GB SQL File ကို ဖြိုခွဲတဲ့ (အရေးကြီးဆုံး အဆင့်)

**လုပ်ရမှာ**: 4GB file ကို တိုက်ရိုက် import လုပ်ခြင်း  
**လုပ်ရမယ်**: Schema (DDL) နဲ့ Data (DML) file နှစ်ခု ခွဲထုတ်ပါ

#### ဘာကြောင့် ခွဲရလဲ?
- Handle လုပ်ဖို့ ပိုလွယ်တယ်
- Schema ကို နှိုင်းယှဉ်ဖို့ လိုတယ်
- Data ကို ငါတို့ ETL process အတွက် သီးခြား လိုတယ်

#### ဘယ်လို ခွဲမလဲ: Python Script

INSERT statements တွေက multi-line ဖြစ်နိုင်တော့ simple grep commands တွေ မလုပ်ဘူး။ Python သုံးမယ်။

**File: `split_sql_dump.py`**

```python
import sys
import os
import traceback

def split_sql_file(large_sql_file, schema_file_path, data_file_path):
    """
    SQL file ကို schema (DDL) နဲ့ data (DML) files နှစ်ခု ခွဲထုတ်တယ်။
    Multi-line INSERT statements တွေကို မှန်မှန်ကန်ကန် handle လုပ်ပေးတယ်။
    """
    print(f"'{large_sql_file}' ကို စတင် process လုပ်နေပါပြီ...")
    
    try:
        # Input: utf-16-le (0xff BOM တွေ့တဲ့ကြောင့်)
        # Output: utf-8 (MySQL compatibility အတွက်)
        with open(large_sql_file, 'r', encoding='utf-16-le') as infile, \
             open(schema_file_path, 'w', encoding='utf-8') as schema_file, \
             open(data_file_path, 'w', encoding='utf-8') as data_file:
            
            is_inside_insert = False
            line_count = 0
            
            print("Processing လုပ်နေတယ်... 4GB file ဆို အချိန်ကြာမယ်။")
            
            for line in infile:
                line_count += 1
                
                # လိုင်း 500,000 တိုင်းမှာ progress update ပြမယ်
                if line_count % 500000 == 0:
                    print(f"Processed {line_count:n} lines...")
                
                stripped_line = line.strip()
                
                if stripped_line.upper().startswith('INSERT INTO'):
                    is_inside_insert = True
                
                if is_inside_insert:
                    data_file.write(line)
                    # INSERT က semicolon နဲ့ ဆုံးတယ်
                    if stripped_line.endswith(';'):
                        is_inside_insert = False
                else:
                    # Schema/setup lines တွေ
                    schema_file.write(line)
            
            print(f"\nပြီးပါပြီ! စုစုပေါင်း {line_count:n} lines process လုပ်ခဲ့တယ်")
            print(f"Schema: {schema_file_path}")
            print(f"Data: {data_file_path}")
    
    except FileNotFoundError:
        print(f"\nError: '{large_sql_file}' ကို ရှာမတွေ့ဘူး။")
    except Exception as e:
        print(f"\nError ဖြစ်ချေ့းတယ်: {e}")
        print("--- Full Traceback ---")
        traceback.print_exc()
        print("----------------------")

if __name__ == "__main__":
    input_file = 'full_old_data_bk.sql'  # မင်းရဲ့ 4GB file
    schema_output = 'schema_old.sql'
    data_output = 'data_old.sql'
    
    if not os.path.exists(input_file):
        print(f"Error: '{input_file}' ကို ရှာမတွေ့ဘူး။")
        print("Filename မှန်ကန်ရဲ့လား ပြန်စစ်ပါ။")
    else:
        split_sql_file(input_file, schema_output, data_output)
```

**ဘယ်လို Run မလဲ:**
```bash
python split_sql_dump.py
```

**ထွက်လာမဲ့ Output:**
- `schema_old.sql` - သေးတဲ့ file (MB အနည်းငယ်) - CREATE TABLE statements တွေ ပါတယ်
- `data_old.sql` - ကြီးတဲ့ file (4GB နီးပါး) - INSERT statements တွေ ပါတယ်

#### ခွဲထားတာကို စစ်ဆေးပါ

**File sizes တွေ ကြည့်ပါ:**
```powershell
# PowerShell မှာ
Get-ChildItem *.sql | Select-Object Name, Length
```

**Schema file စစ်ပါ (CREATE TABLE တွေ့ရမယ်):**
```powershell
Get-Content -Path ".\schema_old.sql" -TotalCount 20
```

**Data file စစ်ပါ (INSERT INTO တွေ့ရမယ်):**
```powershell
Get-Content -Path ".\data_old.sql" -TotalCount 10
Get-Content -Path ".\data_old.sql" -Tail 10
```

### Step 1.3: Schema File မှာ Character Encoding ပြင်ဆင်ခြင်း

**အရေးကြီးဆုံး**: `utf8` ကနေ `utf8mb4` ကို convert လုပ်ရမယ် schema file မှာ။

`schema_old.sql` ကို text editor နဲ့ ဖွင့်ပြီး replace လုပ်ပါ:
- `DEFAULT CHARSET=utf8` → `DEFAULT CHARSET=utf8mb4`
- `CHARSET=utf8` → `CHARSET=utf8mb4`
- `COLLATE=utf8_general_ci` → `COLLATE=utf8mb4_unicode_ci`
- `COLLATE=utf8_unicode_ci` → `COLLATE=utf8mb4_unicode_ci`

**PowerShell သုံးပြီး Automated လုပ်နည်း:**
```powershell
(Get-Content schema_old.sql) -replace 'CHARSET=utf8([^mb4]|$)', 'CHARSET=utf8mb4$1' |
Set-Content schema_old_fixed.sql

(Get-Content schema_old_fixed.sql) -replace 'COLLATE=utf8_', 'COLLATE=utf8mb4_' |
Set-Content schema_old_fixed.sql
```

### Step 1.4: Database အသစ် တည်ဆောက်ခြင်း

AWS Lightsail MySQL မှာ:

```sql
CREATE DATABASE mifos_new_production
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

ပြီးရင် မင်းရဲ့ Mifos application အသစ်ကို run လိုက်ပါ Flyway/Liquibase က schema generate လုပ်ဖို့:
```bash
# Backend ကို start လုပ်
# Flyway က table တွေ auto-create လုပ်ပေးမယ်
```

### Step 1.5: Schema အသစ်ကို Export လုပ်ခြင်း

```bash
mysqldump -u [user] -p --no-data mifos_new_production > schema_new.sql
```

### Step 1.6: DBeaver သုံးပြီး Schemas တွေ နှိုင်းယှဉ်ခြင်း

ဒီမှာ ကွာခြားချက်တွေ အားလုံး ရှာမယ်!

#### Old Schema Database Setup လုပ်ပါ

```sql
CREATE DATABASE mifos_old_schema_test
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Fixed old schema ကို import လုပ်:
```bash
mysql -u [user] -p mifos_old_schema_test < schema_old_fixed.sql
```

#### DBeaver မှာ နှိုင်းယှဉ်ပါ

1. DBeaver ကို ဖွင့်ပါ
2. မင်းရဲ့ Lightsail MySQL ကို connect လုပ်ပါ
3. `mifos_old_schema_test` database ကို Right-click နှိပ်ပါ
4. **Compare → Compare with...** ကို ရွေးပါ
5. Target အနေနဲ့ `mifos_new_production` ကို ရွေးပြီး **Next** နှိပ်ပါ
6. ဘာတွေ နှိုင်းယှဉ်မလဲ မေးမယ်။ **Tables, Columns, Indexes** ကို check ခြစ်ထားပါ
7. **Compare** ကို နှိပ်ပါ
8. Report ကို **HTML** အဖြစ် export လုပ်ပါ

### Step 1.7: HTML Diff ကို Excel ကို ပြောင်းခြင်း

**Excel မှာ:**
1. **Data** tab ကို သွားပါ
2. **From Web** ကို နှိပ်ပါ
3. File path ကို ရိုက်ထည့်ပါ: `file:///D:/MifosMigration/diff-report.html`
4. Comparison table ကို ရွေးပါ
5. **Load** ကို နှိပ်ပါ

### Step 1.8: Schema ကွာခြားချက်တွေ ခွဲခြမ်းစိတ်ဖြာခြင်း

Excel file မှာ formulas တွေ ထည့်ပါ:

**Column C - ကွာခြားချက် မှတ်ပါ:**
```excel
=IF(OR(A1="N/A", B1="N/A"), "", IF(A1<>B1, "diff", ""))
```

**Column E - Table Name ယူပါ:**
```excel
=IF(OR(D9="diff", D9="N/A"), LOOKUP(2, 1/($A$9:$A9="Table Name"), $B$9:$B9), "")
```

**အနီရောင် Highlighting လုပ်ပါ:**
- Column A နဲ့ B ကို select လုပ်ပါ
- **Conditional Formatting → New Rule**
- Formula: `=$C1="diff"`
- Format: Red fill ရွေးပါ

#### Schema Differences တွေမှာ ဘာတွေ ရှာရမလဲ

**အရေးကြီးတဲ့ Changes (ဒါတွေကို Handle လုပ်ရမယ်):**

**1. Data Types ပြောင်းချွေး်းတာ:**
- `tinyint` vs `bit(1)` - Boolean value သိမ်းပုံ ပြောင်းချွေး်းတယ်
  - ဒါက data transformation လုပ်ရမယ် ETL script မှာ
  - MySQL က 0/1 ကို b'0'/b'1' အဖြစ် auto-convert လုပ်ပေးတယ် များသောအားဖြင့်
  
- `decimal` vs `decimal unsigned` - **အရေးကြီးဆုံး!**
  - unsigned က negative numbers တွေ သိမ်းလို့ မရဘူး
  - Old data မှာ negative values တွေ ရှိမရှိ စစ်ရမယ်:
  ```sql
  SELECT * FROM [table_name] WHERE [column_name] < 0;
  ```
  
- `datetime(6)` vs `datetime` - precision ကွာတယ်
  - Microsecond data ပျောက်ချွေး်းမယ် (truncate ဖြစ်မယ်)
  - Client နဲ့ confirm လုပ်ပါ ဒါက acceptable လား

**2. Foreign Key Actions:**
- `CASCADE` vs `RESTRICT` (on delete/update)
  - **အဟောင်း (Cascade)**: Parent record ဖျက်ရင် child records တွေ auto-delete ဖြစ်တယ်
  - **အသစ် (Restrict)**: Parent ကို ဖျက်ခွင့်မပြုဘူး child records တွေ ရှိနေရင်
  - ဒါက business logic အရေးကြီးတဲ့ ပြောင်းလဲမှု ပါ
  - Migration မှာ block မဖြစ်ဘူး (parent tables ကို အရင် import လုပ်ရင်)

**3. Index Types:**
- `UNI` (Unique) vs `MUL` (Multiple/Non-unique)
  - **အဟောင်း (UNI)**: Duplicate values တွေ မရဘူး
  - **အသစ် (MUL)**: Duplicate values တွေ ရတယ်
  - ဒါက migration ကို ပိုလွယ်စေတယ် (duplicates ကြောင့် fail မဖြစ်တော့ဘူး)
  - ဒါပေမဲ့ business rule ပြောင်းချွေး်းတယ်ဆိုတာ client ကို အသိပေးရမယ်

**လျစ်လျူရှုရလို့ရတဲ့ Differences (ပြဿနာ မဟုတ်ဘူး):**
- **Cardinality**: ဂဏန်းတွေ ကွာတာ (5 vs 0, 17 vs 0)
  - Old DB မှာ data ရှိတယ်၊ new DB က empty ဖြစ်နေတယ်
  - Index မှာ unique values ဘယ်နှစ်ခု ရှိလဲ ဆိုတဲ့ estimate ပါ
  - Data import လုပ်ပြီးရင် auto-update ဖြစ်မယ်
  
- **Index Length**: (48K vs 16K)
  - Index က disk space ဘယ်လောက် သုံးနေလဲ
  - Old က data ရှိတော့ 48K၊ new က empty တော့ 16K (default)
  - လျစ်လျူရှုရလို့ရတယ်
  
- **Data Length**: (80K vs 16K)
  - Table က disk space ဘယ်လောက် သုံးလဲ
  - Old က data ရှိတယ်၊ new က empty
  - လျစ်လျူရှုရလို့ရတယ်
  
- **Data Free**: (4M vs 0)
  - DELETE လုပ်ပြီးရင် ကျန်ရစ်တဲ့ fragmented space
  - Old database က live database ဖြစ်တော့ fragmentation ရှိတယ်
  - လျစ်လျူရှုရလို့ရတယ်

- **Display Width**: `tinyint` vs `tinyint(1)`
  - ဒါ နှစ်ခု တူတယ် functionally
  - `(1)` က display width hint ပဲ၊ storage ကို မကန့်သတ်ဘူး
  - လျစ်လျူရှုရလို့ရတယ်

### Step 1.9: Data Mapping Document ရေးဆွဲခြင်း

Diff report ပေါ်မှာတည်ပြီး mapping table တစ်ခု လုပ်ပါ Excel မှာ:

| Old Table | Old Column | New Table | New Column | Transformation Rule | Action Needed |
|-----------|------------|-----------|------------|---------------------|---------------|
| m_client | fullname | m_client | display_name | တိုက်ရိုက်ကူးမယ် | None |
| m_loan | interest_rate | m_loan | interest_rate | DECIMAL(10,2) → DECIMAL(19,6) | Precision စစ်ဆေးပါ |
| m_product | is_active | m_product | is_active | tinyint → bit(1) | 0/1 ကို b'0'/b'1' auto-convert |
| m_client | mobile_no | m_client | mobile_no | UNI → MUL | Duplicates စစ်ဆေးပါ |
| old_table | old_column | N/A | N/A | Table က new မှာ မရှိဘူး | Skip သို့မဟုတ် သီးခြား handle လုပ်ပါ |

**Old Data မှာ စစ်ဆေးရမဲ့ Critical Checks:**

```sql
-- Unsigned columns တွေမှာ negative values တွေ ရှိမရှိ
SELECT * FROM m_loan WHERE interest_rate < 0;

-- New schema က UNIQUE ဖြစ်ပေမဲ့ duplicates တွေ ရှိမရှိ
SELECT mobile_no, COUNT(*) 
FROM m_client 
GROUP BY mobile_no 
HAVING COUNT(*) > 1;

-- NULL values တွေ ရှိမရှိ NOT NULL columns တွေမှာ
SELECT COUNT(*) 
FROM m_client 
WHERE display_name IS NULL;
```

---

## Phase 2: ETL ပတ်ဝန်းကျင် ပြင်ဆင်ခြင်း

အခု ငါတို့ "staging" environment ကို setup လုပ်မယ်။

### Step 2.1: Staging Import DB တည်ဆောက်ခြင်း

ဒါက old data ကို အရင် import လုပ်မဲ့ နေရာပါ။

```sql
CREATE DATABASE mifos_old_import
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;
```

Fixed old schema ကို import လုပ်:
```bash
mysql -u [user] -p mifos_old_import < schema_old_fixed.sql
```

### Step 2.2: Target DB ကို Verify လုပ်ခြင်း

`mifos_new_production` ရှိပြီး new schema ရှိပြီးဆိုတာ သေချာပါ:
```sql
SHOW TABLES FROM mifos_new_production;
```

### Step 2.3: MySQL ကို Large Import အတွက် ပြင်ဆင်ခြင်း

MySQL config (`my.cnf` သို့မဟုတ် `my.ini`) ကို edit လုပ်ပါ:
```ini
[mysqld]
innodb_buffer_pool_size = 2G
innodb_log_file_size = 512M
max_allowed_packet = 256M
```

Config ပြောင်းပြီးရင် MySQL ကို restart လုပ်ပါ။

---

## Phase 3: Data ထုတ်ယူခြင်း၊ ပြောင်းလဲခြင်း၊ ထည့်သွင်းခြင်း (ETL)

ဒါက main migration work ပါ!

### Step 3.1: Old Data ကို Staging DB ထဲ Import လုပ်ခြင်း

`data_old.sql` file ရဲ့ **အပေါ်ဆုံးမှာ** ဒီ lines တွေ ထည့်ပါ:
```sql
SET FOREIGN_KEY_CHECKS=0;
SET UNIQUE_CHECKS=0;
SET SQL_LOG_BIN=0;
SET NAMES utf8mb4;
```

Data ကို import လုပ်:
```bash
mysql -u [user] -p --default-character-set=utf8mb4 mifos_old_import < data_old.sql
```

**Tips for 4GB Import:**
- Import လုပ်နေတုန်း **Tomcat ကို ပိတ်ထားပါ** resource လွတ်ဖို့
- ဒါက 30 မိနစ်ကနေ အချိန်များစွာ ကြာနိုင်တယ်
- Monitor လုပ်ဖို့: `SHOW PROCESSLIST;`

Import ပြီးရင် checks တွေကို ပြန် enable လုပ်ပါ:
```sql
SET FOREIGN_KEY_CHECKS=1;
SET UNIQUE_CHECKS=1;
```

### Step 3.2: ETL Transformation Script ရေးခြင်း

Mapping document ပေါ်မှာတည်ပြီး ETL script တစ်ခု ရေးပါ။

**File: `migrate_data.sql`**

```sql
-- ============================================
-- Mifos Data Migration Script
-- From: mifos_old_import
-- To: mifos_new_production
-- ============================================

SET FOREIGN_KEY_CHECKS=0;

-- ဥပမာ: Clients တွေ Migrate လုပ်ခြင်း
INSERT INTO mifos_new_production.m_client (
    id,
    display_name,
    office_id,
    mobile_no,
    is_active,
    created_date
)
SELECT 
    id,
    fullname,  -- Old column name
    office_id,
    mobile_no,
    CASE WHEN is_active = 1 THEN b'1' ELSE b'0' END,  -- tinyint to bit
    created_date
FROM mifos_old_import.m_client;

-- ဥပမာ: Loans တွေ Migrate လုပ်ခြင်း
INSERT INTO mifos_new_production.m_loan (
    id,
    client_id,
    loan_amount,
    interest_rate,
    is_approved
)
SELECT 
    id,
    client_id,
    loan_amount,
    CAST(interest_rate AS DECIMAL(19,6)),  -- Precision ပြောင်းတာ
    CASE WHEN is_approved = 1 THEN b'1' ELSE b'0' END
FROM mifos_old_import.m_loan;

-- ကျန်တဲ့ tables တွေအတွက် ဆက်ရေးပါ...
-- သတိထား: Parent tables အရင်၊ ပြီးမှ child tables (FK order)

SET FOREIGN_KEY_CHECKS=1;
```

**အရေးကြီးတယ်**: Foreign keys အစဉ်လိုက် မှန်ကန်တဲ့ order နဲ့ လုပ်ပါ:
1. `m_office` (အရင်ဆုံး)
2. `m_staff`
3. `m_client`
4. `m_loan`
5. `m_savings_account`
6. ဆက်ဖြင့်...

**Foreign Key Order ဘယ်လို သိရမလဲ:**
```sql
-- Table တစ်ခုရဲ့ foreign keys တွေ ကြည့်ပါ
SELECT 
    TABLE_NAME,
    COLUMN_NAME,
    REFERENCED_TABLE_NAME,
    REFERENCED_COLUMN_NAME
FROM INFORMATION_SCHEMA.KEY_COLUMN_USAGE
WHERE TABLE_SCHEMA = 'mifos_old_import'
  AND REFERENCED_TABLE_NAME IS NOT NULL
ORDER BY TABLE_NAME;
```

### Step 3.3: User Passwords ကိုင်တွယ်ခြင်း

**အကြံပြုတဲ့ နည်းလမ်း**: Password အားလုံး Reset လုပ်ပါ

```sql
-- User passwords အားလုံးကို default password တစ်ခု ထည့်ပါ
UPDATE mifos_new_production.m_appuser
SET password = '$2a$10$...'; -- 'ChangeMe123' ရဲ့ Bcrypt hash

-- First login မှာ password ပြောင်းခိုင်းပါ
UPDATE mifos_new_production.m_appuser
SET must_change_password = b'1';
```

**Client ကို အသိပေးပါ**: User အားလုံး first login မှာ password ပြောင်းရမယ်။

**ဘာကြောင့် Password Migrate မလုပ်ရသလဲ:**
- Old နဲ့ new Mifos versions တွေက hashing algorithm ကွာနိုင်တယ်
- Security အတွက် password reset လုပ်တာက ပိုကောင်းတယ်
- Migration ပြီးတော့ users တွေ သူတို့ password ပြောင်းလိုက်ရင် ပိုလုံခြုံတယ်

---

## Phase 4: စမ်းသပ်ခြင်းနှင့် အတည်ပြုခြင်း (Testing & Validation)

Data ကူးပြီးပြီဆိုတာနဲ့ မပြီးသေးဘူး! ဒါက အရေးကြီးဆုံး phase တစ်ခုပါ။

### Step 4.1: Row Count Validation လုပ်ခြင်း

```sql
-- အရေးကြီးတဲ့ table တိုင်းမှာ counts တွေ နှိုင်းယှဉ်ပါ
SELECT 'Clients' as Table_Name,
       (SELECT COUNT(*) FROM mifos_old_import.m_client) as Old_Count,
       (SELECT COUNT(*) FROM mifos_new_production.m_client) as New_Count;

SELECT 'Loans' as Table_Name,
       (SELECT COUNT(*) FROM mifos_old_import.m_loan) as Old_Count,
       (SELECT COUNT(*) FROM mifos_new_production.m_loan) as New_Count;

SELECT 'Savings' as Table_Name,
       (SELECT COUNT(*) FROM mifos_old_import.m_savings_account) as Old_Count,
       (SELECT COUNT(*) FROM mifos_new_production.m_savings_account) as New_Count;

-- Major tables အားလုံးအတွက် ထပ်လုပ်ပါ
```

**မျှော်လင့်ထားတဲ့ ရလဒ်**: Old_Count = New_Count ဖြစ်ရမယ် table တိုင်းမှာ

### Step 4.2: Financial Data Validation လုပ်ခြင်း

ဒါက **အရေးကြီးဆုံးပါ** - ငွေကြေး totals တွေ အတိအကျ တူရမယ်!

```sql
-- Loan amounts တွေ validate လုပ်ပါ
SELECT 'Loan Amounts' as Check_Type,
       (SELECT SUM(loan_amount) FROM mifos_old_import.m_loan) as Old_Sum,
       (SELECT SUM(loan_amount) FROM mifos_new_production.m_loan) as New_Sum,
       (SELECT SUM(loan_amount) FROM mifos_old_import.m_loan) - 
       (SELECT SUM(loan_amount) FROM mifos_new_production.m_loan) as Difference;

-- Savings balances တွေ validate လုပ်ပါ
SELECT 'Savings Balances' as Check_Type,
       (SELECT SUM(account_balance) FROM mifos_old_import.m_savings_account) as Old_Sum,
       (SELECT SUM(account_balance) FROM mifos_new_production.m_savings_account) as New_Sum,
       (SELECT SUM(account_balance) FROM mifos_old_import.m_savings_account) - 
       (SELECT SUM(account_balance) FROM mifos_new_production.m_savings_account) as Difference;

-- Transaction totals တွေ validate လုပ်ပါ
SELECT 'Transaction Totals' as Check_Type,
       (SELECT SUM(amount) FROM mifos_old_import.m_payment_detail) as Old_Sum,
       (SELECT SUM(amount) FROM mifos_new_production.m_payment_detail) as New_Sum;
```

**မျှော်လင့်ထားတဲ့ ရလဒ်**: Financial totals တွေ အတိအကျ တူရမယ်! Difference = 0 ဖြစ်ရမယ်။

### Step 4.3: Data Quality Checks လုပ်ခြင်း

```sql
-- Orphaned records စစ်ဆေးပါ (FK violations မရှိရဘူး)
SELECT l.id, l.client_id, 'Loan without Client' as Issue
FROM mifos_new_production.m_loan l
LEFT JOIN mifos_new_production.m_client c ON l.client_id = c.id
WHERE c.id IS NULL;

-- NOT NULL columns တွေမှာ NULL values စစ်ဆေးပါ
SELECT 'Clients with NULL display_name' as Issue, COUNT(*) as Count
FROM mifos_new_production.m_client 
WHERE display_name IS NULL;

-- Required columns တွေမှာ empty strings စစ်ဆေးပါ
SELECT 'Clients with empty display_name' as Issue, COUNT(*) as Count
FROM mifos_new_production.m_client 
WHERE display_name = '';

-- Date columns မှာ invalid dates စစ်ဆေးပါ (ဥပမာ: 1970-01-01)
SELECT 'Loans with default date' as Issue, COUNT(*) as Count
FROM mifos_new_production.m_loan 
WHERE created_date = '1970-01-01';
```

### Step 4.4: Application Testing လုပ်ခြင်း

**အခု real application နဲ့ စမ်းသပ်မယ်:**

**1. Backend ကို New Database ကို Point လုပ်ပါ:**

`application.properties` ကို update လုပ်ပါ:
```properties
spring.datasource.url=jdbc:mysql://[host]:3306/mifos_new_production?useSSL=false&serverTimezone=UTC&useUnicode=true&characterEncoding=UTF-8
spring.datasource.username=[user]
spring.datasource.password=[password]
```

**2. Tomcat ကို New DB connection နဲ့ Restart လုပ်ပါ:**
```bash
systemctl restart tomcat
# သို့မဟုတ်
./catalina.sh stop && ./catalina.sh start
```

**3. Frontend Testing:**
- Reset လုပ်ထားတဲ့ password နဲ့ login ခင်းကြည့်ပါ
- Client list ကြည့်ပါ
- Client တစ်ဦးရဲ့ details ကြည့်ပါ
- Loan details တွေ ကြည့်ပါ
- Transaction history ကြည့်ပါ
- **မြန်မာစာ မှန်မှန် သေချာ ကြည့်ပါ** - `???` မပေါ်ရဘူး

**4. Operations Testing:**
- Loan အသစ် တစ်ခု create လုပ်ကြည့်ပါ (approve မလုပ်ရသေးဘူး)
- Repayment တစ်ခု လုပ်ကြည့်ပါ
- Deposit တစ်ခု လုပ်ကြည့်ပါ
- Report တစ်ခု generate လုပ်ကြည့်ပါ
- Client အသစ် create လုပ်ကြည့်ပါ

**5. Logs တွေ စစ်ဆေးပါ:**
```bash
# Application logs
tail -f /var/log/tomcat/catalina.out

# SQL errors ရှာပါ
tail -f /var/log/tomcat/catalina.out | grep -i "error"

# MySQL logs
tail -f /var/log/mysql/error.log
```

**ဘာတွေ ရှာရမလဲ:**
- SQL errors တွေ
- Character encoding issues (မြန်မာစာမှာ `???` ပေါ်တာ)
- Foreign key violations
- Data type mismatches
- Slow query warnings

---

## Phase 5: Production Go-Live (တကယ် Run ခြင်း)

Testing အောင်မြင်ပြီဆိုရင် real migration အတွက် prepare လုပ်မယ်။

### Step 5.1: Client နဲ့ Downtime စီစဉ်နှိုင်းနှိုင်းခြင်း

**Client ကို အသိပေးရမဲ့ အချက်တွေ:**
- **ဘယ်တော့**: Migration လုပ်မဲ့ ရက်နဲ့ အချိန်
- **ဘယ်လောက်ကြာမလဲ**: 2-4 နာရီ downtime ခန့်မှန်းထားပါ
- **ဘာတွေ ဖြစ်မလဲ**: System အဟောင်း offline ဖြစ်မယ်၊ migration ပြီးရင် system အသစ် သုံးလို့ရမယ်
- **Users ဘာလုပ်ရမလဲ**: User အားလုံး first login မှာ password ပြောင်းရမယ်
- **Data backup**: Migration မလုပ်ခင် နောက်ဆုံး backup တစ်ခု လုပ်ပေးဖို့ တောင်းပါ

### Step 5.2: နောက်ဆုံး Data Backup ယူခြင်း

**Migration ရက်မှာ:**
1. Client ကို system အဟောင်း ကို ရပ်ခိုင်းပါ (stop using)
2. Client က နောက်ဆုံး backup create လုပ်ပါ:
   ```bash
   mysqldump [old_db] > final_backup_$(date +%Y%m%d).sql
   ```
3. Client က file ပို့ပေးပါမယ်

### Step 5.3: Final Migration Run လုပ်ခြင်း

**Databases တွေ Drop လုပ်ပြီး ပြန်လုပ်ပါ:**
```sql
DROP DATABASE IF EXISTS mifos_old_import;
DROP DATABASE IF EXISTS mifos_new_production;

CREATE DATABASE mifos_old_import 
CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

CREATE DATABASE mifos_new_production 
CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**Scripts တွေ အစဉ်လိုက် Run လုပ်ပါ:**
1. Final backup file ကို split လုပ်ပါ (Python script သုံးပြီး)
2. Schema file မှာ encoding fix လုပ်ပါ
3. Old schema ကို staging DB မှာ import လုပ်ပါ
4. New Mifos run လုပ်ပြီး new schema generate လုပ်ပါ
5. Old data ကို staging DB မှာ import လုပ်ပါ
6. ETL script run လုပ်ပြီး data migrate လုပ်ပါ
7. Validation queries အားလုံး run လုပ်ပါ

### Step 5.4: Final Validation Checklist

- [ ] Row counts အားလုံး တူတယ်
- [ ] Financial totals အားလုံး တူတယ်
- [ ] Orphaned records မရှိဘူး
- [ ] Application က errors မရှိဘဲ start ဖြစ်တယ်
- [ ] Reset password နဲ့ login ခင်းလို့ ရတယ်
- [ ] Client data အားလုံး ကြည့်လို့ ရတယ်
- [ ] Loan data အားလုံး ကြည့်လို့ ရတယ်
- [ ] မြန်မာစာ မှန်မှန်ကန်ကန် ပြတယ်
- [ ] Transaction အသစ် create လုပ်လို့ ရတယ်
- [ ] Reports တွေ မှန်မှန်ကန်ကန် generate ဖြစ်တယ်

### Step 5.5: Go Live!

**Validation အားလုံး pass ဖြစ်ရင်:**
1. Production Tomcat ကို `mifos_new_production` သုံးအောင် update လုပ်ပါ
2. Tomcat ကို restart လုပ်ပါ
3. Angular frontend ကို restart လုပ်ပါ
4. Client ကို system အသစ် သုံးလို့ရပြီလို့ အသိပေးပါ
5. Default password နဲ့ instructions တွေ ပို့ပေးပါ

**Client ကို ပို့ပေးရမဲ့ Information:**
```
Subject: Mifos System Migration - အောင်မြင်ပါပြီ

ကျွန်တော်တို့ Mifos system migration ကို အောင်မြင်စွာ ပြီးမြောက်ပါပြီ။

အသစ် System URL: https://your-new-system.com
Default Password: ChangeMe123

Login ခင်းတဲ့အခါ password အသစ် ပြောင်းဖို့ တောင်းပါလိမ့်မယ်။

မေးခွန်း ရှိရင် ဆက်သွယ်ပေးပါ။
```

---

## Phase 6: Post-Migration (Migration အပြီး)

### Step 6.1: Monitoring လုပ်ခြင်း (ပထမ 48 နာရီ)

**Logs တွေကို အမြဲ စောင့်ကြည့်ပါ:**
```bash
# Application logs
tail -f /var/log/tomcat/catalina.out | grep -i error

# MySQL logs
tail -f /var/log/mysql/error.log

# MySQL process list
watch -n 5 'mysql -u root -p -e "SHOW PROCESSLIST"'

# Disk space စစ်ပါ
df -h
```

**ဘာတွေ Monitor လုပ်ရမလဲ:**
- Slow queries တွေ
- Character encoding errors
- Foreign key violations
- Application crashes
- High CPU/Memory usage
- Disk space running out

### Step 6.2: ချက်ချင်း Backup လုပ်ခြင်း

Migration အောင်မြင်ပြီဆိုတာနဲ့ backup ယူပါ!

```bash
# Compressed backup
mysqldump -u [user] -p mifos_new_production | gzip > mifos_migration_success_$(date +%Y%m%d).sql.gz

# S3 ကို upload လုပ်ပါ
aws s3 cp mifos_migration_success_*.sql.gz s3://your-backup-bucket/
```

### Step 6.3: Cleanup လုပ်ခြင်း

**အရာအားလုံး အဆင်ပြေတယ်ဆို confirm ဖြစ်ပြီးရင် (1 ပတ်အကြာမှာ):**

```sql
-- Staging database ဖျက်ပါ
DROP DATABASE mifos_old_import;

-- Schema test database ဖျက်ပါ
DROP DATABASE mifos_old_schema_test;
```

**File အဟောင်းတွေ Archive လုပ်ပါ:**
```bash
# Archive folder လုပ်ပါ
mkdir archive
mv full_old_data_bk.sql archive/
mv schema_old.sql archive/
mv data_old.sql archive/

# S3 Glacier ကို upload လုပ်ပါ long-term storage အတွက်
aws s3 cp archive/ s3://your-archive-bucket/ --recursive --storage-class GLACIER
```

### Step 6.4: Documentation ရေးခြင်း

**Migration Report တစ်ခု လုပ်ပါ:**
- စုစုပေါင်း records ဘယ်နှစ်ခု migrate လုပ်ခဲ့လဲ
- ဘယ် data transformations တွေ လုပ်ခဲ့လဲ
- Migrate လုပ်လို့ မရတဲ့ data တွေ ရှိလား (အကြောင်းရင်းတွေနဲ့)
- သိထားရမဲ့ issues သို့မဟုတ် warnings တွေ
- Post-migration configuration changes တွေ

**ဥပမာ Migration Report Format:**
```markdown
# Mifos Data Migration Report
Date: 2025-10-21
Performed by: [Your Team]

## Migration Statistics
- Total Clients Migrated: 5,234
- Total Loans Migrated: 12,456
- Total Savings Accounts: 8,901
- Total Transactions: 45,678

## Data Transformations Applied
1. Character encoding: utf8 → utf8mb4
2. Boolean fields: tinyint → bit(1)
3. Interest rates: DECIMAL(10,2) → DECIMAL(19,6)
4. User passwords: Reset to default (must change on first login)

## Issues Encountered
1. Found 3 negative values in interest_rate column
   - Action: Set to 0.00 after client confirmation
2. Found 12 duplicate mobile numbers
   - Action: Allowed (new schema permits duplicates)

## Post-Migration Configuration
- All users must change password on first login
- Foreign key behavior changed from CASCADE to RESTRICT
- Backup schedule: Daily at 2:00 AM

## Validation Results
✅ Row counts match: All tables verified
✅ Financial totals match: Exact match confirmed
✅ Application testing: All features working
✅ Character encoding: Myanmar text displays correctly
```

---

## အရေးကြီးတဲ့ သတိပေးချက်တွေနှင့် သတိထားရမဲ့အချက်များ

### ⚠️ မဖြစ်မနေ လုပ်ရမဲ့အရာများ (MUST-DO Items)

- **Character Encoding**: အရာအားလုံး `utf8mb4` ဖြစ်ရမယ်၊ `utf8` မဟုတ်ဘူး
- **File ခွဲထုတ်ပါ**: 4GB ကို တိုက်ရိုက် import ဘယ်တော့မှ မလုပ်ပါနဲ့
- **Schema နှိုင်းယှဉ်ပါ**: DBeaver သုံးပါ၊ text diff မသုံးပါနဲ့
- **Staging Database**: Staging မှာ အရင် import လုပ်ပြီးမှ production ကို ETL လုပ်ပါ
- **Foreign Key Order**: Parent tables တွေ အရင်၊ child tables နောက်မှ import လုပ်ပါ
- **Password Reset**: Hashed passwords တွေ migrate လုပ်ဖို့ မကြိုးစားပါနဲ့
- **အရာအားလုံး Validate လုပ်ပါ**: Row counts နဲ့ financial totals အတိအကျ တူရမယ်

### ❌ ဘယ်တော့မှ မလုပ်ရ (DON'T DO)

- Old DDL ကို new database ကို တိုက်ရိုက် import မလုပ်ပါနဲ့
- Online "HTML to Excel Converter" websites တွေ မသုံးပါနဲ့ (sensitive data နဲ့ security risk)
- Validation phase ကို မကျော်ပါနဲ့
- Testing မလုပ်ဘဲ go live မလုပ်ပါနဲ့
- Migration ပြီးတာနဲ့ ချက်ချင်း backup ယူဖို့ မမေ့ပါနဲ့
- Client ကို inform မလုပ်ဘဲ migration မစပါနဲ့

### 💡 Pro Tips (အသုံးတင်တဲ့ အကြံပြုချက်တွေ)

- **Lightsail Resources**: Import လုပ်နေတုန်း Tomcat ပိတ်ထားပါ RAM/CPU လွတ်ဖို့
- **JDBC Connection**: `characterEncoding=UTF-8` ကို connection string မှာ ထည့်ဖို့ သေချာပါ
- **Progress Tracking**: Python script က progress ပြတယ် - နှေးနေရင် စိတ်ရှည်ပါ
- **Test Environment**: ဖြစ်နိုင်ရင် သီးခြား Lightsail instance တစ်ခုမှာ test run အရင်လုပ်ပါ
- **Client Communication**: အရမ်း communicate လုပ်ပါ! တွေ့လာမှုတိုင်းမှာ အသိပေးပါ
- **Dry Run**: Production migration မလုပ်ခင် test data နဲ့ အရင် လေ့ကျင့်ပါ
- **Documentation**: လုပ်သမျှ step တိုင်းကို မှတ်တမ်းတင်ပါ - နောက်ပြန်ကြည့်ရင် အသုံးတင်တယ်
- **Backup Everything**: Migration မစခင် backup၊ ပြီးတာနဲ့ backup၊ အရာအားလုံး backup!

---

## Troubleshooting (ပြဿနာဖြေရှင်းခြင်း)

### Issue 1: Import က "Out of Memory" ဖြစ်တယ်

**Symptoms:**
```
ERROR: MySQL server has gone away
ERROR: Out of memory
```

**အဖြေ:**
```bash
# 1. Tomcat ကို ရပ်ပါ resource လွတ်ဖို့
systemctl stop tomcat

# 2. MySQL config ပြင်ပါ
sudo nano /etc/mysql/my.cnf
```

Config မှာ ဒါတွေ ပြင်ပါ:
```ini
[mysqld]
max_allowed_packet = 512M
innodb_buffer_pool_size = 2G
innodb_log_file_size = 512M
```

```bash
# 3. MySQL restart လုပ်ပါ
sudo systemctl restart mysql

# 4. Import ပြန်လုပ်ပါ
```

### Issue 2: Character Encoding မှာ `???` ပေါ်တယ်

**Symptoms:**
မြန်မာစာတွေက `???` သို့မဟုတ် garbled characters တွေ ပြတယ်

**စစ်ဆေးပါ:**
```sql
-- Database charset စစ်ပါ
SHOW CREATE DATABASE mifos_new_production;
-- Result မှာ utf8mb4 ပါရမယ်

-- Table charset စစ်ပါ
SHOW CREATE TABLE m_client;
-- Result မှာ DEFAULT CHARSET=utf8mb4 ပါရမယ်

-- Connection charset စစ်ပါ
SHOW VARIABLES LIKE 'character%';
```

**အဖြေ:**
1. JDBC connection string မှာ `characterEncoding=UTF-8` ရှိမရှိ စစ်ပါ
2. All tables က `utf8mb4` သုံးရဲ့လား verify လုပ်ပါ
3. Import လုပ်တုန်း `--default-character-set=utf8mb4` သုံးရဲ့လား စစ်ပါ

### Issue 3: Foreign Key Constraint ကြောင့် Fail ဖြစ်တယ်

**Symptoms:**
```
ERROR: Cannot add or update a child row: a foreign key constraint fails
```

**အဖြေ:**
```sql
-- 1. Import order မှန်ရဲ့လား စစ်ပါ (parent tables အရင်လား)
-- 2. Parent records တွေ အားလုံး ရှိပြီးလား စစ်ပါ

-- Orphaned records ရှာပါ
SELECT l.id, l.client_id
FROM mifos_old_import.m_loan l
LEFT JOIN mifos_old_import.m_client c ON l.client_id = c.id
WHERE c.id IS NULL;

-- 3. အောက်က setting နဲ့ import လုပ်ပါ
SET FOREIGN_KEY_CHECKS=0;
-- ... your INSERT statements ...
SET FOREIGN_KEY_CHECKS=1;
```

### Issue 4: Row Counts မတူဘူး

**Symptoms:**
Old database မှာ 5,000 records ရှိတယ်၊ new database မှာ 4,950 ပဲ ရှိတယ်

**စစ်ဆေးပါ:**
```bash
# MySQL error log ကြည့်ပါ
tail -f /var/log/mysql/error.log

# Duplicate key violations ရှာပါ
grep -i "duplicate" /var/log/mysql/error.log

# Constraint violations ရှာပါ
grep -i "constraint" /var/log/mysql/error.log
```

**အဖြေ:**
```sql
-- Missing records ရှာပါ
SELECT o.id 
FROM mifos_old_import.m_client o
LEFT JOIN mifos_new_production.m_client n ON o.id = n.id
WHERE n.id IS NULL;

-- ဘာကြောင့် skip ဖြစ်ချွေး်းလဲ သိဖို့ ပိုစစ်ပါ
```

### Issue 5: Application က Migration နောက် Start မဖြစ်ဘူး

**Symptoms:**
Tomcat logs မှာ SQL errors တွေ ပြတယ်

**စစ်ဆေးပါ:**
```bash
# Tomcat logs ကြည့်ပါ
tail -100 /var/log/tomcat/catalina.out

# SQL errors ရှာပါ
grep -i "sql" /var/log/tomcat/catalina.out | grep -i "error"
```

**အဖြေ:**
1. Database connection string မှန်ရဲ့လား verify လုပ်ပါ
2. Required tables အားလုံး ရှိလား စစ်ပါ:
```sql
SHOW TABLES FROM mifos_new_production;
```
3. Schema version table (Flyway/Liquibase) စစ်ပါ:
```sql
SELECT * FROM flyway_schema_history ORDER BY installed_rank DESC LIMIT 5;
```

### Issue 6: Migration က အရမ်းနှေးတယ် (4GB က 8 နာရီထက် ကြာတယ်)

**အကြောင်းရင်းများ:**
- Indexes တွေ အများကြီး ရှိတယ်
- Foreign key checks enabled ဖြစ်နေတယ်
- Lightsail instance က သေးတယ် (RAM/CPU မလောက်ဘူး)

**အဖြေ:**
```sql
-- Import မစခင် ဒါတွေ disable လုပ်ပါ
SET FOREIGN_KEY_CHECKS=0;
SET UNIQUE_CHECKS=0;
SET AUTOCOMMIT=0;
SET sql_log_bin=0;

-- Import ပြီးရင် enable ပြန်လုပ်ပါ
COMMIT;
SET FOREIGN_KEY_CHECKS=1;
SET UNIQUE_CHECKS=1;
SET AUTOCOMMIT=1;
```

သို့မဟုတ် Lightsail instance ကို ယာယီ upgrade လုပ်ပါ migration အတွက်။

---

## Team Communication Plan (Team ဆက်သွယ်ရေး အစီအစဉ်)

### Daily Standup (Migration ကာလအတွင်း)

**တိုင်းနေ့ မနက် 9:00 AM မှာ:**

**Report လုပ်ရမဲ့အရာများ:**
- လက်ရှိ phase နဲ့ progress (ဥပမာ: "Phase 3 - 60% complete")
- Blockers သို့မဟုတ် issues တွေ ရှိလား
- Row count validation results
- Data quality concerns တွေ ရှိလား
- ဒီနေ့ ဘာတွေ လုပ်မလဲ

**ဥပမာ Update:**
```
Phase: 3 (ETL)
Progress: 75% - m_client, m_loan, m_savings_account migrated
Row Counts: ✅ All match
Financial Totals: ✅ All match
Blockers: None
Today: Complete remaining tables, start Phase 4 testing
```

### Escalation Path (ပြဿနာတက်လာရင် ဘယ်လို report လုပ်မလဲ)

**Level 1 - Minor Issues:** Team အတွင်းမှာ ဖြေရှင်းလို့ရတဲ့ issues
- Schema differences ကို ETL နဲ့ handle လုပ်လို့ရတာတွေ
- Configuration issues
- နည်းပညာပိုင်း အသေးအဖွဲ့တွေ

**Level 2 - Medium Issues:** Client ကို confirm လုပ်ရမဲ့ issues
- Data inconsistencies
- Business rule changes
- Negative values, duplicates စသည့် data quality issues

**Level 3 - Major Issues:** Project ဆိုင်ရာလောက်တဲ့ issues
- Schema changes လိုတယ်
- Migration approach ပြောင်းရမယ်
- Timeline delay လိုတယ်

### Communication Channels (ဆက်သွယ်ရေး နည်းလမ်းများ)

**Team Internal:**
- Slack/Teams: Daily updates, quick questions
- Email: Formal reports, documentation sharing
- Video Calls: Complex discussions, troubleshooting sessions

**Client Communication:**
- Email: Official communications, confirmations
- Video Calls: Weekly progress updates, issue discussions
- Phone: Urgent issues only

---

## Success Criteria (အောင်မြင်မှု သတ်မှတ်ချက်များ)

Migration က အောင်မြင်တယ်လို့ ပြောနိုင်တာက:

### Technical Success Criteria

- ✅ **Data Integrity**: Data အားလုံး error မရှိဘဲ import ဖြစ်တယ်
- ✅ **Row Counts Match**: Table တိုင်းမှာ row counts အတိအကျ တူတယ်
- ✅ **Financial Accuracy**: Financial totals တွေ အတိအကျ တူတယ် (အရေးကြီးဆုံး!)
- ✅ **No Orphaned Records**: Foreign key relationships အားလုံး intact ဖြစ်နေတယ်
- ✅ **Application Stability**: Application က errors မရှိဘဲ run နေတယ်
- ✅ **Character Encoding**: မြန်မာစာ မှန်မှန်ကန်ကန် display ဖြစ်တယ်
- ✅ **Performance**: System က acceptable speed နဲ့ run နေတယ်

### Functional Success Criteria

- ✅ **User Login**: Users တွေ login ခင်းနိုင်တယ် (password reset လုပ်ပြီး)
- ✅ **View Data**: Client data, loan data အားလုံး ကြည့်နိုင်တယ်
- ✅ **Create Transactions**: Transaction အသစ်တွေ create လုပ်လို့ရတယ်
- ✅ **Reports**: Reports တွေ မှန်မှန်ကန်ကန် generate ဖြစ်တယ်
- ✅ **CRUD Operations**: Create, Read, Update, Delete လုပ်လို့ရတယ်

### Business Success Criteria

- ✅ **Client Confirmation**: Client က data accuracy ကို confirm လုပ်ပြီး
- ✅ **User Acceptance**: End users တွေက system အသစ် သုံးလို့ရတယ်လို့ confirm လုပ်ပြီး
- ✅ **System Stability**: 48 နာရီ stable ဖြစ်နေတယ် major issues မရှိဘဲ
- ✅ **Backup Secured**: Production data backup လုံခြုံစွာ သိမ်းဆည်းပြီး
- ✅ **Documentation Complete**: Migration documentation ပြီးပြည့်စုံပြီး

---

## Rollback Plan (နောက်ပြန်ဆုတ်ဖို့ အစီအစဉ်)

Migration က fail ဖြစ်ချွေး်းရင် ဘယ်လို နောက်ပြန်ဆုတ်မလဲ?

### ဘယ်တော့ Rollback လုပ်ရမလဲ?

**Critical Issues တွေ တွေ့ရင်:**
- Financial totals မတူဘူး
- Data corruption ဖြစ်နေတယ်
- Application က အလုပ် လုံးချ မလုပ်ဘူး
- Client က data ကို မလက်ခံဘူး

### Rollback Steps

**1. Client အဟောင်း System ကို ပြန်ဖွင့်ပါ:**
```bash
# Client ဆီမှာ
# Database အဟောင်း ကို ပြန် restore လုပ်ပါ
# Application အဟောင်း ကို ပြန် start လုပ်ပါ
```

**2. Client ကို အသိပေးပါ:**
```
Subject: Migration Rollback - ယာယီ ပြန်ဆုတ်ထားပါတယ်

တွေ့လာရေး problems တွေ တွေ့လို့ system အဟောင်းကို ပြန်သုံးပါ။
သင်း data က လုံခြုံပါတယ်။
ပြဿနာ fix လုပ်ပြီး ပြန်စမယ်။
```

**3. Root Cause Analysis လုပ်ပါ:**
- ဘာကြောင့် fail ဖြစ်ချွေး်းလဲ သေချာ စဉ်းစားပါ
- ဘယ် step မှာ မှားချွေး်းလဲ
- ဘယ်လို ပြင်ရမလဲ

**4. Fix နဲ့ Retry:**
- ပြဿနာ fix လုပ်ပါ
- Test environment မှာ ပြန်စမ်းပါ
- အောင်မြင်ရင် production မှာ retry လုပ်ပါ

---

## မေးခွန်းတွေ ရှိလား? ပြဿနာတွေ တွေ့လား?

**ဒီ plan မှာ မပါတဲ့ ပြဿနာတွေ တွေ့ရင်:**

**Document လုပ်ပါ:**
1. **တာလုပ်နေတုန်းလဲ** - ဘယ် step? ဘယ် command?
2. **အတိအကျ error message** - screenshot ရိုက်ပါ
3. **ဘာတွေ စမ်းပြီးပြီလဲ** - ကြိုးစားတဲ့တဲ့ solutions တွေ
4. **Environment details** - MySQL version, Lightsail specs, etc.

**Team ကို ချက်ချင်း ပြောပါ:**
- Slack/Teams မှာ post လုပ်ပါ သို့မဟုတ်
- Video call တောင်ပါ urgent ဆိုရင်
- တစ်ယောက်တည်း struggle မလုပ်ပါနဲ့!

**သတိထားရမဲ့ အရာများ:**
- အသေးစိတ် မှတ်တမ်းတင်ပါ (screenshots, logs)
- ဘာမှ မဖျောက်အောင် backup များများ ယူပါ
- မသေချာရင် မလုပ်သေးပါနဲ့ - အရင် မေးပါ
- Test environment မှာ အရင် စမ်းပါ production မလုပ်ခင်

---

## အပိတ် မှတ်ချက်

Team လေးရေ! 👋

ဒီ migration က ကြီးမားတဲ့ project တစ်ခု ဖြစ်ပေမဲ့ ငါတို့ plan မှန်မှန်ကန်ကန် လုပ်ပြီး၊ phase တစ်ခုတိုင်းစီ သေချာစွာ လုပ်ချွေး်းရင် အောင်မြင်နိုင်ပါတယ်။

**အရေးကြီးဆုံး အချက်များ:**
- 🐢 **သည်းခံပါ** - အလျင်မလုပ်ပါနဲ့၊ validation တိုင်းကို သေချာ လုပ်ပါ
- ✅ **Validate Everything** - Row counts, financial totals, character encoding
- 💬 **Communicate** - Team နဲ့ မြဲမြဲ update လုပ်ပါ၊ client နဲ့ မြဲမြဲ ဆက်သွယ်ပါ
- 💾 **Backup Everything** - လုပ်သမျှ အရာတိုင်းကို backup ယူပါ
- 📝 **Document Everything** - လုပ်သမျှ ရေးမှတ်ပါ၊ နောက်မှ အသုံးတင်မယ်

ဒီ plan က roadmap ပါ။ တစ်ခုမှားရင် သို့မဟုတ် မသေချာရင်:
1. ရပ်ပါ (Stop)
2. သေချာစစ်ပါ (Verify)  
3. မေးပါ (Ask)
4. ပြီးမှ ဆက်လုပ်ပါ (Then proceed)

**Final Reminders:**
- 4GB file ကို တိုက်ရိုက် import ဘယ်တော့မှ မလုပ်ပါနဲ့
- Character encoding က `utf8mb4` သာ သုံးပါ
- Financial totals တွေ အတိအကျ တူရမယ် - ဒါက negotiable မဟုတ်ဘူး
- Client နဲ့ အမြဲ communicate လုပ်ပါ
- Test ကောင်းကောင်းမလုပ်ဘဲ production မသွားပါနဲ့

Good luck team! ငါတို့ လုပ်နိုင်ပါတယ်! 💪🔥

---

## Appendix: Quick Reference Commands

### MySQL Commands
```sql
-- Database သတင်းအချက်အလက် ကြည့်ခြင်း
SHOW DATABASES;
SHOW TABLES FROM database_name;
SHOW CREATE TABLE table_name;

-- Character set စစ်ဆေးခြင်း
SHOW VARIABLES LIKE 'character%';
SHOW VARIABLES LIKE 'collation%';

-- Process list ကြည့်ခြင်း
SHOW PROCESSLIST;
SHOW FULL PROCESSLIST;

-- Table size ကြည့်ခြင်း
SELECT 
    table_name AS 'Table',
    ROUND(((data_length + index_length) / 1024 / 1024), 2) AS 'Size (MB)'
FROM information_schema.TABLES
WHERE table_schema = 'database_name'
ORDER BY (data_length + index_length) DESC;
```

### Backup & Restore Commands
```bash
# Full backup
mysqldump -u user -p database_name > backup.sql

# Schema only
mysqldump -u user -p --no-data database_name > schema.sql

# Data only
mysqldump -u user -p --no-create-info database_name > data.sql

# Compressed backup
mysqldump -u user -p database_name | gzip > backup.sql.gz

# Restore
mysql -u user -p database_name < backup.sql

# Restore compressed
gunzip < backup.sql.gz | mysql -u user -p database_name
```

### Server Status Commands
```bash
# Disk space
df -h

# Memory usage
free -h

# CPU usage
top

# MySQL status
systemctl status mysql

# Tomcat status
systemctl status tomcat

# Check MySQL logs
tail -f /var/log/mysql/error.log

# Check Tomcat logs
tail -f /var/log/tomcat/catalina.out
```

---

**Plan Version**: 2.0  
**Last Updated**: 2025-10-21  
**Language**: Myanmar (Burmese)  
**Maintained by**: [Your Team Name]

*ဒီ plan ကို လိုအပ်သလို update လုပ်ပြီး၊ မင်းတို့ specific requirements အတွက် customize လုပ်နိုင်ပါတယ်။*

---

## ဆက်စပ် Resources

### Documentation Links
- MySQL 8.0 Documentation: https://dev.mysql.com/doc/
- Mifos Documentation: https://docs.mifos.org/
- DBeaver Documentation: https://dbeaver.com/docs/
- AWS Lightsail Documentation: https://docs.aws.amazon.com/lightsail/

### Tools Required
1. **DBeaver** - Database comparison and management
2. **Python 3.x** - For splitting large SQL files
3. **MySQL Client** - For command-line operations
4. **Text Editor** - VS Code, Notepad++, or similar
5. **Excel/LibreOffice Calc** - For mapping documentation

### Team Contacts
```
Project Manager: [Name] - [Email] - [Phone]
Lead Developer: [Name] - [Email] - [Phone]
Database Admin: [Name] - [Email] - [Phone]
Client Contact: [Name] - [Email] - [Phone]
```

---

**🎯 Remember: Take your time, validate everything, and communicate constantly. Success is in the details!**