# 🔄 Kaggle Leaderboard Sync Workflow

## Visual Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CELERY BEAT SCHEDULER                        │
│                      (Triggers every 5 minutes)                      │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  sync_kaggle_leaderboard_auto()                      │
│                         Celery Task                                  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│         Query: Competition.objects.filter(is_active=True,            │
│                      kaggle_competition_id__isnull=False)            │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
                        ┌────────────────┐
                        │ For each       │
                        │ Competition    │
                        └────────┬───────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                KaggleLeaderboardSync.                                │
│              sync_competition_leaderboard()                          │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                ┌────────────────┼────────────────┐
                ▼                ▼                ▼
         ┌──────────┐     ┌──────────┐    ┌──────────┐
         │  Fetch   │     │ Process  │    │ Cleanup  │
         └──────────┘     └──────────┘    └──────────┘
                │                │                │
                │                │                │
                ▼                ▼                ▼
```

## Detailed Step-by-Step Process

### 1. FETCH PHASE (fetch_and_save_to_csv)

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1.1: Create Temp Directory                             │
│ → mkdir temp_kaggle_data/{slug}_{timestamp}/                │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1.2: Execute Kaggle CLI                                │
│ → kaggle competitions leaderboard {slug} --download         │
│   --path temp_kaggle_data/{slug}_{timestamp}/               │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1.3: Kaggle API Downloads ZIP                          │
│ → {slug}.zip (contains full leaderboard CSV)                │
│   Size: ~50KB for 1,744 entries                             │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1.4: Extract ZIP                                       │
│ → Extracts: {slug}-publicleaderboard-{timestamp}.csv        │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1.5: Fix Windows Path Issues                           │
│ → If filename contains ':' (timestamp)                      │
│   Rename: 2025-10-31T16:26:26 → 2025-10-31_T_16_26_26      │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1.6: Delete ZIP File                                   │
│ → Remove {slug}.zip to save space                           │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1.7: Return CSV Path                                   │
│ → Returns: Path to cleaned CSV file                         │
└─────────────────────────────────────────────────────────────┘
```

### 2. PROCESS PHASE (process_csv_and_update_db)

```
┌─────────────────────────────────────────────────────────────┐
│ Step 2.1: Read CSV with Pandas                              │
│ → df = pd.read_csv(csv_path)                                │
│   Columns: Rank, TeamId, TeamName, LastSubmissionDate,      │
│            Score, SubmissionCount, TeamMemberUserNames       │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2.2: Initialize Counters                               │
│ → created_count = 0                                          │
│ → updated_count = 0                                          │
│ → skipped_count = 0                                          │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
               ┌───────────────┐
               │ For each row  │
               │ in DataFrame  │
               └───────┬───────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2.3: Extract Data from Row                             │
│ → team_name = row['TeamName']                               │
│ → rank = row['Rank']                                        │
│ → score = float(row['Score'])                               │
│ → submission_date = row['LastSubmissionDate']               │
│ → team_usernames = row['TeamMemberUserNames']               │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2.4: Match User (Priority Order)                       │
│ 1. Try each username in TeamMemberUserNames                 │
│    → For username in team_usernames.split(','):             │
│      → Try: User.objects.get(username=username)             │
│                                                              │
│ 2. If not found, try TeamName                               │
│    → Try: User.objects.get(username=team_name)              │
│                                                              │
│ 3. If still not found                                       │
│    → Skip this entry, increment skipped_count               │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
              ┌────────────────┐
              │ User found?    │
              └────┬──────┬────┘
                   │ Yes  │ No
                   ▼      ▼
                  Yes   Skip
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2.5: Update or Create LeaderboardEntry                 │
│ → LeaderboardEntry.objects.update_or_create(                │
│     competition=competition,                                 │
│     user=user,                                               │
│     defaults={                                               │
│       'score': score,                                        │
│       'rank': rank,                                          │
│       'kaggle_team_name': team_name,                         │
│       'submission_date': parsed_date                         │
│     }                                                        │
│   )                                                          │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
              ┌────────────────┐
              │ Was created or │
              │    updated?    │
              └────┬──────┬────┘
                   │      │
            Created│      │Updated
                   ▼      ▼
            created_count++  updated_count++
                       │
                       ▼
             ┌─────────────────┐
             │ More rows?      │
             └────┬──────┬─────┘
                  │ Yes  │ No
                  │      │
                  └──────┴─────▶ Continue to Step 2.6
                       
┌─────────────────────────────────────────────────────────────┐
│ Step 2.6: Log Summary                                       │
│ → INFO: Database update complete                            │
│         Created: {created_count}                             │
│         Updated: {updated_count}                             │
│         Skipped: {skipped_count}                             │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2.7: Return Total Processed                            │
│ → Returns: created_count + updated_count                     │
└─────────────────────────────────────────────────────────────┘
```

### 3. CLEANUP PHASE (cleanup_csv)

```
┌─────────────────────────────────────────────────────────────┐
│ Step 3.1: Delete CSV File                                   │
│ → os.remove(csv_path)                                       │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3.2: Delete Parent Directory                           │
│ → os.rmdir(parent_directory)                                │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3.3: Log Cleanup                                       │
│ → INFO: Cleaned up CSV: {filename}                          │
└─────────────────────────────────────────────────────────────┘
```

## Complete Example: Spaceship Titanic

```
Time: 00:00:00
┌──────────────────────────────────────────────────────┐
│ Celery Beat: Time to run sync!                      │
└──────────────────────────────────────────────────────┘
                       │
Time: 00:00:01         ▼
┌──────────────────────────────────────────────────────┐
│ Task: sync_kaggle_leaderboard_auto() started        │
│ Found: 1 active competition (spaceship-titanic)     │
└──────────────────────────────────────────────────────┘
                       │
Time: 00:00:02         ▼
┌──────────────────────────────────────────────────────┐
│ FETCH: Downloading spaceship-titanic leaderboard    │
│ Command: kaggle competitions leaderboard            │
│          spaceship-titanic --download                │
└──────────────────────────────────────────────────────┘
                       │
Time: 00:00:10         ▼
┌──────────────────────────────────────────────────────┐
│ Downloaded: spaceship-titanic.zip (58KB)            │
│ Extracted: spaceship-titanic-publicleaderboard-...  │
│ Renamed to: ...2025-10-31_T_16_26_26.csv            │
│ Result: 1,744 entries                                │
└──────────────────────────────────────────────────────┘
                       │
Time: 00:00:11         ▼
┌──────────────────────────────────────────────────────┐
│ PROCESS: Reading CSV with 1,744 entries             │
│ Columns: Rank, TeamId, TeamName, LastSubmission...  │
└──────────────────────────────────────────────────────┘
                       │
Time: 00:00:11-13      ▼
┌──────────────────────────────────────────────────────┐
│ Processing rows...                                   │
│ Row 1: CADang → User found → Updated                │
│ Row 2: नरेश कोपरखैरणे → User not found → Skipped   │
│ Row 3: Jeki Wan Taufik → User found → Created       │
│ ...                                                  │
│ Row 1744: BrandonFT → User not found → Skipped      │
└──────────────────────────────────────────────────────┘
                       │
Time: 00:00:13         ▼
┌──────────────────────────────────────────────────────┐
│ Database update complete:                            │
│ - Created: 12 new entries                            │
│ - Updated: 8 existing entries                        │
│ - Skipped: 1,724 (users not in system)              │
└──────────────────────────────────────────────────────┘
                       │
Time: 00:00:14         ▼
┌──────────────────────────────────────────────────────┐
│ CLEANUP: Deleting CSV and temp directory            │
│ Removed: All temporary files                         │
└──────────────────────────────────────────────────────┘
                       │
Time: 00:00:15         ▼
┌──────────────────────────────────────────────────────┐
│ Task complete!                                       │
│ Result: "Synced 20 entries across 1 competition"    │
└──────────────────────────────────────────────────────┘
                       │
Time: 00:05:00         ▼
┌──────────────────────────────────────────────────────┐
│ Celery Beat: Time to run sync again!                │
│ (Process repeats every 5 minutes)                   │
└──────────────────────────────────────────────────────┘
```

## Error Handling Flow

```
Any Step
    │
    ├─▶ Exception Caught
    │       │
    │       ▼
    │   Log Error
    │   (logger.error)
    │       │
    │       ▼
    │   Return None or Skip
    │       │
    │       ▼
    │   Continue to Next Competition
    │
    └─▶ Success
            │
            ▼
        Continue Workflow
```

## Data Flow Diagram

```
┌─────────────┐
│   Kaggle    │
│     API     │
└──────┬──────┘
       │ 1. CLI Download
       ▼
┌─────────────┐
│  ZIP File   │
│  (58KB)     │
└──────┬──────┘
       │ 2. Extract
       ▼
┌─────────────┐
│  CSV File   │
│ (1,744 rows)│
└──────┬──────┘
       │ 3. Parse
       ▼
┌─────────────┐
│  DataFrame  │
│  (pandas)   │
└──────┬──────┘
       │ 4. Process Each Row
       ▼
┌─────────────┐         ┌─────────────┐
│ User Match  │────────▶│   User DB   │
└──────┬──────┘         └─────────────┘
       │ 5. Update
       ▼
┌─────────────┐         ┌─────────────┐
│Leaderboard  │────────▶│ Leaderboard │
│   Entry     │         │     DB      │
└──────┬──────┘         └─────────────┘
       │ 6. Cleanup
       ▼
┌─────────────┐
│  Delete     │
│   CSV       │
└─────────────┘
```

## Performance Timeline (Typical Run)

```
Action                          Time        Cumulative
─────────────────────────────────────────────────────
Task Start                      0.0s        0.0s
Find Active Competitions        0.1s        0.1s
Create Temp Directory           0.1s        0.2s
Execute Kaggle CLI              8.0s        8.2s
Extract ZIP                     0.3s        8.5s
Rename CSV (Windows fix)        0.1s        8.6s
Delete ZIP                      0.1s        8.7s
Read CSV into DataFrame         0.5s        9.2s
Process 1,744 rows              3.5s       12.7s
  ├─ Row parsing               (1.0s)
  ├─ User lookups              (2.0s)
  └─ Database updates          (0.5s)
Delete CSV                      0.1s       12.8s
Delete temp directory           0.1s       12.9s
Log and return                  0.1s       13.0s
─────────────────────────────────────────────────────
Total Time                                 ~13 seconds
```

## Memory Usage

```
Component                   Memory      Notes
──────────────────────────────────────────────────────
CSV File (disk)            ~200 KB     Temporary
ZIP File (disk)            ~58 KB      Deleted quickly
DataFrame (RAM)            ~500 KB     During processing
Database connections       ~5 MB       Persistent
Python process overhead    ~50 MB      Celery worker
──────────────────────────────────────────────────────
Peak Memory Usage          ~56 MB      Minimal footprint
```

## Concurrency Model

```
┌─────────────────────────────────────────────────────────┐
│                  Celery Beat Scheduler                  │
│              (1 instance, triggers tasks)               │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                    Celery Worker Pool                   │
│                  (Multiple workers)                     │
└────────────────────────┬────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
    Worker 1         Worker 2         Worker 3
        │                │                │
        ▼                ▼                ▼
Competition 1    Competition 2    Competition 3
(Sequential)     (Sequential)     (Sequential)

Note: Each competition is processed sequentially within
      its worker to avoid race conditions.
```

## Success Indicators

```
✅ Download Complete
   └─▶ INFO: Downloaded complete leaderboard: 1744 entries

✅ Processing Complete
   └─▶ INFO: CSV has 1744 entries with columns: [...]

✅ Database Updated
   └─▶ INFO: Database update complete - Created: X, Updated: Y, Skipped: Z

✅ Cleanup Complete
   └─▶ INFO: Cleaned up CSV: {filename}

✅ Task Complete
   └─▶ INFO: Synced {total} entries for {slug}
```

---

**This workflow runs automatically every 5 minutes, keeping your leaderboard perfectly synced with Kaggle!**
