# Shahrazad Final Architecture & Staging Baseline Specification

**Document Version:** Final Architecture & Baseline (Post-Audit)
**Audit Execution Timestamp:** 2026-11-03T17:40:55.849013+00:00
**Database Engine:** PostgreSQL 17.6
**Connected Database / Role:** postgres / postgres (Timezone: UTC)
**Project Reference:** Unconfirmed (Pending confirmation from Supabase Dashboard)
**Status:** Architectural Specification & Staging Execution Plan (No Migration / No Schema Execution on Production)

---

## Executive Summary

This specification establishes the authoritative, empirical baseline for the "Shahrazad Interactive Storyteller" subsystem, synthesizing the actual schema extracted via `Shahrazad_Staging_ReadOnly_Audit.csv` with the architectural requirements established in earlier design rounds (V1–V4.1).

### Key Empirical Findings from Staging Audit
1. **Engine Baseline**: PostgreSQL 17.6 running on standard Supabase infrastructure.
2. **Current Schema Scope**: Exactly 9 base tables in `public` schema comprising 103 columns.
3. **Absence of Story Subsystem**: Zero `story_*` tables, zero story-related RPC functions, zero story indexes, and no `pages.story_source_change_token` column exist currently.
4. **Page Status Domain**: Allowed values strictly constrained by check constraint `pages_status_check` to: `'processing'`, `'ocr_review_required'`, `'ready'`, `'failed'`.
5. **Page Numbering**: Index `pages_book_idx` on `(book_id, page_number)` is a standard non-unique B-tree index. Serialization is handled logically by the existing `create_page_with_next_number` function which acquires a row lock on `books` (`FOR UPDATE`).
6. **Storage State**: Exactly one private bucket exists: `page-images`. There is no dedicated audio bucket (`story-audio`).
7. **Full-Text Search (FTS)**: 29 standard FTS configurations in `pg_catalog` (including `arabic`, `english`, `simple`), but **no Polish dictionary or configuration exists** in PostgreSQL. Polish text tokenization must utilize `simple` or normalized tokens.
8. **Security & Permissions**: RLS is enabled on all 9 public tables. However, table grants reveal excessive privileges: both `anon` and `authenticated` roles currently possess `ALL` permissions (`DELETE, INSERT, REFERENCES, SELECT, TRIGGER, TRUNCATE, UPDATE`) on all 9 public tables. Since RLS does not protect against `TRUNCATE`, privilege remediation is critical.


---

## Section 1: Staging Baseline Inventory & CSV Evidence
The tables below represent the exact ground-truth state of the Staging database extracted during the read-only audit on 2026-11-03T17:40:55Z.

### 1.1 Installed Extensions
| Extension | Schema | Version |
| :--- | :--- | :--- |
| `pg_stat_statements` | `extensions` | `1.11` |
| `pgcrypto` | `extensions` | `1.3` |
| `plpgsql` | `pg_catalog` | `1.0` |
| `supabase_vault` | `vault` | `0.3.1` |
| `uuid-ossp` | `extensions` | `1.1` |

### 1.2 Public Tables & Column Architecture
| Table | Columns Count | Primary Key | Key Foreign Keys |
| :--- | :--- | :--- | :--- |
| `books` | 8 | `books_pkey` | None |
| `difficult_words` | 13 | `difficult_words_pkey` | `difficult_words_book_id_fkey -> books`, `difficult_words_page_id_fkey -> pages` |
| `lesson_events` | 8 | `lesson_events_pkey` | `lesson_events_page_id_fkey -> pages`, `lesson_events_session_id_fkey -> reading_sessions` |
| `pages` | 17 | `pages_pkey` | `pages_book_id_fkey -> books` |
| `profiles` | 6 | `profiles_pkey` | None |
| `reading_attempts` | 11 | `reading_attempts_pkey` | `reading_attempts_page_id_fkey -> pages`, `reading_attempts_session_id_fkey -> reading_sessions` |
| `reading_sessions` | 15 | `reading_sessions_pkey` | `reading_sessions_book_id_fkey -> books`, `reading_sessions_page_id_fkey -> pages` |
| `review_attempts` | 11 | `review_attempts_pkey` | `review_attempts_book_id_fkey -> books` |
| `settings` | 14 | `settings_pkey` | None |

#### Detailed Column Breakdown by Table

##### Table: `public.books`
| # | Column | Type | Nullable | Default |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `id` | `uuid` | NO | `gen_random_uuid()` |
| 2 | `user_id` | `uuid` | NO | `auth.uid()` |
| 3 | `title` | `text` | NO | `NULL` |
| 4 | `author` | `text` | YES | `NULL` |
| 5 | `cover_path` | `text` | YES | `NULL` |
| 6 | `language` | `text` | NO | `'pl'::text` |
| 7 | `created_at` | `timestamp with time zone` | NO | `now()` |
| 8 | `updated_at` | `timestamp with time zone` | NO | `now()` |

##### Table: `public.difficult_words`
| # | Column | Type | Nullable | Default |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `id` | `uuid` | NO | `gen_random_uuid()` |
| 2 | `user_id` | `uuid` | NO | `auth.uid()` |
| 3 | `book_id` | `uuid` | YES | `NULL` |
| 4 | `page_id` | `uuid` | YES | `NULL` |
| 5 | `word` | `text` | NO | `NULL` |
| 6 | `note` | `text` | YES | `NULL` |
| 7 | `times_missed` | `integer` | NO | `1` |
| 8 | `needs_review` | `boolean` | NO | `true` |
| 9 | `created_at` | `timestamp with time zone` | NO | `now()` |
| 10 | `updated_at` | `timestamp with time zone` | NO | `now()` |
| 11 | `issue_type` | `text` | NO | `'pronunciation'::text` |
| 12 | `chunk_index` | `integer` | YES | `NULL` |
| 13 | `last_seen_at` | `timestamp with time zone` | NO | `now()` |

##### Table: `public.lesson_events`
| # | Column | Type | Nullable | Default |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `id` | `uuid` | NO | `gen_random_uuid()` |
| 2 | `user_id` | `uuid` | NO | `auth.uid()` |
| 3 | `session_id` | `uuid` | NO | `NULL` |
| 4 | `page_id` | `uuid` | NO | `NULL` |
| 5 | `chunk_index` | `integer` | NO | `NULL` |
| 6 | `event_type` | `text` | NO | `NULL` |
| 7 | `payload` | `jsonb` | NO | `'{}'::jsonb` |
| 8 | `created_at` | `timestamp with time zone` | NO | `now()` |

##### Table: `public.pages`
| # | Column | Type | Nullable | Default |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `id` | `uuid` | NO | `gen_random_uuid()` |
| 2 | `user_id` | `uuid` | NO | `auth.uid()` |
| 3 | `book_id` | `uuid` | NO | `NULL` |
| 4 | `page_number` | `integer` | NO | `1` |
| 5 | `image_path` | `text` | YES | `NULL` |
| 6 | `status` | `text` | NO | `'processing'::text` |
| 7 | `ocr_text` | `text` | YES | `NULL` |
| 8 | `lesson` | `jsonb` | YES | `NULL` |
| 9 | `ocr_confidence` | `text` | YES | `NULL` |
| 10 | `is_completed` | `boolean` | NO | `false` |
| 11 | `error_detail` | `text` | YES | `NULL` |
| 12 | `created_at` | `timestamp with time zone` | NO | `now()` |
| 13 | `updated_at` | `timestamp with time zone` | NO | `now()` |
| 14 | `completed_manually` | `boolean` | NO | `false` |
| 15 | `actual_reading_seconds` | `integer` | NO | `0` |
| 16 | `chunks_completed_count` | `integer` | NO | `0` |
| 17 | `last_read_at` | `timestamp with time zone` | YES | `NULL` |

##### Table: `public.profiles`
| # | Column | Type | Nullable | Default |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `id` | `uuid` | NO | `NULL` |
| 2 | `parent_name` | `text` | YES | `NULL` |
| 3 | `child_nickname` | `text` | NO | `'بطلة'::text` |
| 4 | `onboarded` | `boolean` | NO | `false` |
| 5 | `created_at` | `timestamp with time zone` | NO | `now()` |
| 6 | `updated_at` | `timestamp with time zone` | NO | `now()` |

##### Table: `public.reading_attempts`
| # | Column | Type | Nullable | Default |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `id` | `uuid` | NO | `gen_random_uuid()` |
| 2 | `user_id` | `uuid` | NO | `auth.uid()` |
| 3 | `session_id` | `uuid` | NO | `NULL` |
| 4 | `page_id` | `uuid` | NO | `NULL` |
| 5 | `chunk_index` | `integer` | NO | `NULL` |
| 6 | `target_text` | `text` | NO | `NULL` |
| 7 | `completed` | `boolean` | NO | `false` |
| 8 | `comprehension_status` | `text` | YES | `NULL` |
| 9 | `correction_summary` | `text` | YES | `NULL` |
| 10 | `transcript` | `text` | YES | `NULL` |
| 11 | `created_at` | `timestamp with time zone` | NO | `now()` |

##### Table: `public.reading_sessions`
| # | Column | Type | Nullable | Default |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `id` | `uuid` | NO | `gen_random_uuid()` |
| 2 | `user_id` | `uuid` | NO | `auth.uid()` |
| 3 | `page_id` | `uuid` | NO | `NULL` |
| 4 | `book_id` | `uuid` | NO | `NULL` |
| 5 | `state` | `jsonb` | NO | `'{}'::jsonb` |
| 6 | `lesson_status` | `text` | NO | `'PAGE_READY'::text` |
| 7 | `current_chunk_index` | `integer` | NO | `0` |
| 8 | `completed` | `boolean` | NO | `false` |
| 9 | `interruptions` | `integer` | NO | `0` |
| 10 | `duration_seconds` | `integer` | NO | `0` |
| 11 | `summary` | `jsonb` | YES | `NULL` |
| 12 | `started_at` | `timestamp with time zone` | NO | `now()` |
| 13 | `ended_at` | `timestamp with time zone` | YES | `NULL` |
| 14 | `updated_at` | `timestamp with time zone` | NO | `now()` |
| 15 | `chunk_timings` | `jsonb` | YES | `'[]'::jsonb` |

##### Table: `public.review_attempts`
| # | Column | Type | Nullable | Default |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `id` | `uuid` | NO | `gen_random_uuid()` |
| 2 | `user_id` | `uuid` | NO | `auth.uid()` |
| 3 | `book_id` | `uuid` | YES | `NULL` |
| 4 | `page_ids` | `ARRAY` | NO | `'{}'::uuid[]` |
| 5 | `scope` | `text` | NO | `'page'::text` |
| 6 | `total_right` | `integer` | NO | `0` |
| 7 | `total_questions` | `integer` | NO | `0` |
| 8 | `section_scores` | `jsonb` | NO | `'{}'::jsonb` |
| 9 | `missed_words` | `ARRAY` | NO | `'{}'::text[]` |
| 10 | `mastered_words` | `ARRAY` | NO | `'{}'::text[]` |
| 11 | `created_at` | `timestamp with time zone` | NO | `now()` |

##### Table: `public.settings`
| # | Column | Type | Nullable | Default |
| :--- | :--- | :--- | :--- | :--- |
| 1 | `user_id` | `uuid` | NO | `NULL` |
| 2 | `reading_level` | `text` | NO | `'A1'::text` |
| 3 | `correction_strictness` | `text` | NO | `'gentle'::text` |
| 4 | `comprehension_frequency` | `text` | NO | `'every_chunk'::text` |
| 5 | `voice_speed` | `text` | NO | `'slow'::text` |
| 6 | `tutor_behavior` | `text` | NO | `'ask_to_explain'::text` |
| 7 | `vad_silence_ms` | `integer` | NO | `1800` |
| 8 | `vad_prefix_padding_ms` | `integer` | NO | `300` |
| 9 | `vad_end_sensitivity` | `text` | NO | `'low'::text` |
| 10 | `live_voice` | `text` | NO | `'Aoede'::text` |
| 11 | `mock_mode` | `boolean` | NO | `false` |
| 12 | `save_full_transcripts` | `boolean` | NO | `false` |
| 13 | `reduced_motion` | `boolean` | NO | `false` |
| 14 | `updated_at` | `timestamp with time zone` | NO | `now()` |

### 1.3 Constraints & Foreign Keys
| Table | Constraint Name | Type | Validated | Definition |
| :--- | :--- | :--- | :--- | :--- |
| `books` | `books_pkey` | PRIMARY KEY | True | `PRIMARY KEY (id)` |
| `books` | `books_title_check` | CHECK | True | `CHECK (((length(title) >= 1) AND (length(title) <= 200)))` |
| `difficult_words` | `difficult_words_book_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (book_id) REFERENCES books(id) ON DELETE CASCADE` |
| `difficult_words` | `difficult_words_issue_type_check` | CHECK | True | `CHECK ((issue_type = ANY (ARRAY['pronunciation'::text, 'comprehension'::text])))` |
| `difficult_words` | `difficult_words_page_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (page_id) REFERENCES pages(id) ON DELETE CASCADE` |
| `difficult_words` | `difficult_words_pkey` | PRIMARY KEY | True | `PRIMARY KEY (id)` |
| `lesson_events` | `lesson_events_page_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (page_id) REFERENCES pages(id) ON DELETE CASCADE` |
| `lesson_events` | `lesson_events_pkey` | PRIMARY KEY | True | `PRIMARY KEY (id)` |
| `lesson_events` | `lesson_events_session_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (session_id) REFERENCES reading_sessions(id) ON DELETE CASCADE` |
| `pages` | `pages_book_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (book_id) REFERENCES books(id) ON DELETE CASCADE` |
| `pages` | `pages_pkey` | PRIMARY KEY | True | `PRIMARY KEY (id)` |
| `pages` | `pages_status_check` | CHECK | True | `CHECK ((status = ANY (ARRAY['processing'::text, 'ocr_review_required'::text, 'ready'::text, 'failed'::text])))` |
| `profiles` | `profiles_pkey` | PRIMARY KEY | True | `PRIMARY KEY (id)` |
| `reading_attempts` | `reading_attempts_page_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (page_id) REFERENCES pages(id) ON DELETE CASCADE` |
| `reading_attempts` | `reading_attempts_pkey` | PRIMARY KEY | True | `PRIMARY KEY (id)` |
| `reading_attempts` | `reading_attempts_session_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (session_id) REFERENCES reading_sessions(id) ON DELETE CASCADE` |
| `reading_sessions` | `reading_sessions_book_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (book_id) REFERENCES books(id) ON DELETE CASCADE` |
| `reading_sessions` | `reading_sessions_page_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (page_id) REFERENCES pages(id) ON DELETE CASCADE` |
| `reading_sessions` | `reading_sessions_pkey` | PRIMARY KEY | True | `PRIMARY KEY (id)` |
| `review_attempts` | `review_attempts_book_id_fkey` | FOREIGN KEY | True | `FOREIGN KEY (book_id) REFERENCES books(id) ON DELETE CASCADE` |
| `review_attempts` | `review_attempts_pkey` | PRIMARY KEY | True | `PRIMARY KEY (id)` |
| `settings` | `settings_comprehension_frequency_check` | CHECK | True | `CHECK ((comprehension_frequency = ANY (ARRAY['every_chunk'::text, 'every_2_chunks'::text])))` |
| `settings` | `settings_correction_strictness_check` | CHECK | True | `CHECK ((correction_strictness = ANY (ARRAY['gentle'::text, 'normal'::text])))` |
| `settings` | `settings_pkey` | PRIMARY KEY | True | `PRIMARY KEY (user_id)` |
| `settings` | `settings_reading_level_check` | CHECK | True | `CHECK ((reading_level = ANY (ARRAY['A1'::text, 'A2'::text])))` |
| `settings` | `settings_tutor_behavior_check` | CHECK | True | `CHECK ((tutor_behavior = ANY (ARRAY['ask_to_explain'::text, 'explain_when_asked'::text])))` |
| `settings` | `settings_vad_end_sensitivity_check` | CHECK | True | `CHECK ((vad_end_sensitivity = ANY (ARRAY['low'::text, 'high'::text])))` |
| `settings` | `settings_vad_prefix_padding_ms_check` | CHECK | True | `CHECK (((vad_prefix_padding_ms >= 0) AND (vad_prefix_padding_ms <= 2000)))` |
| `settings` | `settings_vad_silence_ms_check` | CHECK | True | `CHECK (((vad_silence_ms >= 300) AND (vad_silence_ms <= 6000)))` |
| `settings` | `settings_voice_speed_check` | CHECK | True | `CHECK ((voice_speed = ANY (ARRAY['slow'::text, 'normal'::text])))` |

### 1.4 Existing Indexes
| Table | Index Definition |
| :--- | :--- |
| `books` | `CREATE UNIQUE INDEX books_pkey ON public.books USING btree (id)` |
| `books` | `CREATE INDEX books_user_idx ON public.books USING btree (user_id)` |
| `difficult_words` | `CREATE INDEX difficult_words_book_page_idx ON public.difficult_words USING btree (user_id, book_id, page_id, needs_review)` |
| `difficult_words` | `CREATE UNIQUE INDEX difficult_words_pkey ON public.difficult_words USING btree (id)` |
| `difficult_words` | `CREATE UNIQUE INDEX difficult_words_user_page_word_type_uidx ON public.difficult_words USING btree (user_id, page_id, word, issue_type)` |
| `difficult_words` | `CREATE INDEX difficult_words_user_type_idx ON public.difficult_words USING btree (user_id, issue_type, needs_review)` |
| `difficult_words` | `CREATE INDEX words_user_idx ON public.difficult_words USING btree (user_id)` |
| `lesson_events` | `CREATE INDEX events_session_idx ON public.lesson_events USING btree (session_id, created_at)` |
| `lesson_events` | `CREATE UNIQUE INDEX lesson_events_pkey ON public.lesson_events USING btree (id)` |
| `pages` | `CREATE INDEX idx_pages_completion_audit ON public.pages USING btree (book_id, is_completed, completed_manually)` |
| `pages` | `CREATE INDEX pages_book_idx ON public.pages USING btree (book_id, page_number)` |
| `pages` | `CREATE UNIQUE INDEX pages_pkey ON public.pages USING btree (id)` |
| `pages` | `CREATE INDEX pages_user_idx ON public.pages USING btree (user_id)` |
| `profiles` | `CREATE UNIQUE INDEX profiles_pkey ON public.profiles USING btree (id)` |
| `reading_attempts` | `CREATE INDEX attempts_session_idx ON public.reading_attempts USING btree (session_id)` |
| `reading_attempts` | `CREATE UNIQUE INDEX reading_attempts_pkey ON public.reading_attempts USING btree (id)` |
| `reading_sessions` | `CREATE UNIQUE INDEX reading_sessions_pkey ON public.reading_sessions USING btree (id)` |
| `reading_sessions` | `CREATE INDEX sessions_page_idx ON public.reading_sessions USING btree (page_id)` |
| `reading_sessions` | `CREATE INDEX sessions_user_idx ON public.reading_sessions USING btree (user_id, updated_at DESC)` |
| `review_attempts` | `CREATE UNIQUE INDEX review_attempts_pkey ON public.review_attempts USING btree (id)` |
| `review_attempts` | `CREATE INDEX review_attempts_user_created_idx ON public.review_attempts USING btree (user_id, created_at DESC)` |
| `settings` | `CREATE UNIQUE INDEX settings_pkey ON public.settings USING btree (user_id)` |

### 1.5 Database Functions & Ownership
| Function Signature | Owner | Security Definer | search_path | Volatility |
| :--- | :--- | :--- | :--- | :--- |
| `create_page_with_next_number(uuid,text,text,text,text,jsonb)` | `postgres` | False | `['search_path=public']` | `v` |
| `handle_new_user()` | `postgres` | True | `['search_path=public']` | `v` |
| `touch_updated_at()` | `postgres` | False | `['search_path=public']` | `v` |

### 1.6 Active Triggers
| Table | Trigger Name | Timing & Event | Target Function |
| :--- | :--- | :--- | :--- |
| `books` | `t_books` | `CREATE TRIGGER t_books BEFORE UPDATE ON public.books FOR EACH ROW EXECUTE FUNCTION touch_updated_at()` | `touch_updated_at` |
| `difficult_words` | `t_words` | `CREATE TRIGGER t_words BEFORE UPDATE ON public.difficult_words FOR EACH ROW EXECUTE FUNCTION touch_updated_at()` | `touch_updated_at` |
| `pages` | `t_pages` | `CREATE TRIGGER t_pages BEFORE UPDATE ON public.pages FOR EACH ROW EXECUTE FUNCTION touch_updated_at()` | `touch_updated_at` |
| `profiles` | `t_profiles` | `CREATE TRIGGER t_profiles BEFORE UPDATE ON public.profiles FOR EACH ROW EXECUTE FUNCTION touch_updated_at()` | `touch_updated_at` |
| `reading_sessions` | `t_sessions` | `CREATE TRIGGER t_sessions BEFORE UPDATE ON public.reading_sessions FOR EACH ROW EXECUTE FUNCTION touch_updated_at()` | `touch_updated_at` |
| `settings` | `t_settings` | `CREATE TRIGGER t_settings BEFORE UPDATE ON public.settings FOR EACH ROW EXECUTE FUNCTION touch_updated_at()` | `touch_updated_at` |

### 1.7 Row Level Security (RLS) & Existing Policies
| Table | Policy Name | Command | Roles | Permissive | Using Clause / Check Clause |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `public.books` | `own books` | ALL | `authenticated` | PERMISSIVE | `USING: (user_id = auth.uid())` / `CHECK: (user_id = auth.uid())` |
| `public.difficult_words` | `own words` | ALL | `authenticated` | PERMISSIVE | `USING: (user_id = auth.uid())` / `CHECK: (user_id = auth.uid())` |
| `public.lesson_events` | `own events` | ALL | `authenticated` | PERMISSIVE | `USING: (user_id = auth.uid())` / `CHECK: (user_id = auth.uid())` |
| `public.pages` | `own pages` | ALL | `authenticated` | PERMISSIVE | `USING: (user_id = auth.uid())` / `CHECK: (user_id = auth.uid())` |
| `public.profiles` | `own profile` | ALL | `authenticated` | PERMISSIVE | `USING: (id = auth.uid())` / `CHECK: (id = auth.uid())` |
| `public.reading_attempts` | `own attempts` | ALL | `authenticated` | PERMISSIVE | `USING: (user_id = auth.uid())` / `CHECK: (user_id = auth.uid())` |
| `public.reading_sessions` | `own sessions` | ALL | `authenticated` | PERMISSIVE | `USING: (user_id = auth.uid())` / `CHECK: (user_id = auth.uid())` |
| `public.review_attempts` | `own review attempts` | ALL | `authenticated` | PERMISSIVE | `USING: (user_id = auth.uid())` / `CHECK: (user_id = auth.uid())` |
| `public.settings` | `own settings` | ALL | `authenticated` | PERMISSIVE | `USING: (user_id = auth.uid())` / `CHECK: (user_id = auth.uid())` |
| `storage.objects` | `own page images delete` | DELETE | `authenticated` | PERMISSIVE | `USING: ((bucket_id = 'page-images'::text) AND ((storage.foldername(name))[1] = (auth.uid())::text))` / `CHECK: None` |
| `storage.objects` | `own page images insert` | INSERT | `authenticated` | PERMISSIVE | `USING: None` / `CHECK: ((bucket_id = 'page-images'::text) AND ((storage.foldername(name))[1] = (auth.uid())::text))` |
| `storage.objects` | `own page images read` | SELECT | `authenticated` | PERMISSIVE | `USING: ((bucket_id = 'page-images'::text) AND ((storage.foldername(name))[1] = (auth.uid())::text))` / `CHECK: None` |

### 1.8 Storage Infrastructure
| Bucket ID | Name | Public | File Size Limit | Allowed MIME Types |
| :--- | :--- | :--- | :--- | :--- |
| `page-images` | `page-images` | False | None | All |

### 1.9 Full-Text Search (FTS) Configurations Inventory
PostgreSQL pg_catalog currently provides 29 dictionaries/configurations:
`arabic`, `armenian`, `basque`, `catalan`, `danish`, `dutch`, `english`, `finnish`, `french`, `german`, `greek`, `hindi`, `hungarian`, `indonesian`, `irish`, `italian`, `lithuanian`, `nepali`, `norwegian`, `portuguese`, `romanian`, `russian`, `serbian`, `simple`, `spanish`, `swedish`, `tamil`, `turkish`, `yiddish`.
**Empirical Evidence:** Polish (`polish`) is absent. Any attempt to execute `to_tsvector('polish', ...)` or `plainto_tsquery('polish', ...)` will raise a database runtime exception (`text search configuration "polish" does not exist`).

---

## Section 2: Gap Analysis & V4/V4.1 Reconciliation

A critical review comparing the assumptions of drafts V4 and V4.1 against the empirical Staging audit reveals essential discrepancies that must be reconciled.

| Area | V4 / V4.1 Assumption | Staging Ground Truth | Reconciliation Action | Decision Classification |
| :--- | :--- | :--- | :--- | :--- |
| **PostgreSQL Version** | Assumed PG 15 or 16 generic | PostgreSQL 17.6 | Leverage PG 17 native features safely; verify syntax compatibility. | **[Empirical Staging Evidence]** |
| **Existing Tables** | Assumed hypothetical `story_%` tables might exist or partially conflict | Zero `story_%` tables exist in public schema | Clean greenfield migration for story subsystem, but must guard existing tables. | **[Empirical Staging Evidence]** |
| **`pages` Change Tracking** | Assumed `story_source_change_token` existed | Column is absent | Add `story_source_change_token text NOT NULL DEFAULT gen_random_uuid()` via safe migration. | **[Empirical Staging Evidence]** |
| **`page_number` Uniqueness** | Assumed UNIQUE constraint on `(book_id, page_number)` | Standard non-unique B-tree index `pages_book_idx`; concurrency controlled by `create_page_with_next_number` with `FOR UPDATE` | Do NOT add premature UNIQUE constraint without deduplication check. Rely on existing locking pattern or add unique constraint only after deduplication script. | **[Design Decision Grounded in Staging]** |
| **Story Audio Storage** | Assumed storage bucket `story-audio` existed or could share `page-images` | Only `page-images` exists; sharing would violate mime isolation and ACLs | Create dedicated private bucket `story-audio` via Supabase API / SQL with restrictive RLS and signed URLs. | **[Design Decision Grounded in Staging]** |
| **Polish FTS** | Assumed `to_tsvector('polish', text)` | Polish configuration does not exist in pg_catalog | Use `simple` configuration for Polish search, coupled with case/accent folding and canonical keyword matching. | **[Empirical Staging Evidence]** |
| **Table Grants (Security)** | Assumed standard least-privilege Supabase grants | `anon` and `authenticated` roles hold `TRUNCATE, TRIGGER, REFERENCES` on all 9 public tables! | Revoke `TRUNCATE, TRIGGER, REFERENCES` immediately; retain DML only where app requires, protect completions strictly. | **[Empirical Staging Evidence]** |
| **Story Invalidation Source** | Ambiguity between `ocr_text` vs `lesson` | `pages` has both `ocr_text` (raw) and `lesson` (jsonb chunks) | Story narration is generated strictly from the validated semantic story text (`ocr_text` / chunk sequence). Invalidation token changes if `ocr_text` or `lesson` changes. | **[Design Decision Grounded in Staging]** |


---

## Section 3: Classified Architectural Decisions

Each architectural choice is strictly classified under one of three categories:
1. **[Empirical Staging Evidence]**: Backed directly by data from the 2026-11-03 audit CSV.
2. **[Design Decision Grounded in Staging]**: Architectural pattern established to satisfy product constraints without introducing unneeded infrastructure.
3. **[Pending Verification Gate]**: Requires confirmation on live Staging environment before production deployment.

### Decision 3.1: Database-Backed Queue (PostgreSQL Native)
- **Classification**: **[Design Decision Grounded in Staging]**
- **Rationale**: No Redis, RabbitMQ, or Celery instances are available or desired. With PostgreSQL 17.6 and existing load profiles, transactional `SKIP LOCKED` job queues provide atomic claim, fencing token validation, and complete observability without external operational overhead.

### Decision 3.2: Separate Private Audio Bucket (`story-audio`)
- **Classification**: **[Design Decision Grounded in Staging]**
- **Rationale**: Staging currently only has `page-images` (private). Mixing audio MP3 artifacts with OCR page images in the same bucket compromises lifecycle policies, file-size enforcement, and storage RLS. A distinct `story-audio` bucket must be created as `public = false`.

### Decision 3.3: Multilingual Search Strategy Without Native Polish FTS
- **Classification**: **[Empirical Staging Evidence]**
- **Rationale**: Audit confirmed `polish` FTS does not exist in `pg_catalog`. The system will support search across Arabic, Polish, and English by:
  - Using `arabic` configuration for Arabic queries/docs.
  - Using `english` configuration for English queries/docs.
  - Using `simple` configuration for Polish queries/docs, indexing un-stemmed lowercased word tokens.
  - In addition, storing localized segment summaries and Polish keyword tokens in `fts_doc`.

### Decision 3.4: Server-Side Playback Checkpointing & Strict Anti-Spoofing
- **Classification**: **[Design Decision Grounded in Staging]**
- **Rationale**: A client cannot be trusted to mark a segment completed simply by sending `p_position_seconds >= duration`. Completion requires:
  1. Active listening session validation.
  2. Sequential server-validated checkpoints where `position` progresses in monotonic alignment with elapsed real time (`last_checkpoint_at` + elapsed server seconds).
  3. Strict matching of `(session_id, book_id, segment_id, language)`.
  4. Ensuring source pages are validated and un-invalidated.

### Decision 3.5: No-Spoilers Anti-Hallucination Semantic RAG
- **Classification**: **[Design Decision Grounded in Staging]**
- **Rationale**: The Shahrazad persona must answer child questions strictly using scenes the child has *actually completed listening to*. Completed segments are queried via RAG filtered strictly by `story_completions.completed = true` for the active user, preventing plot spoilers and hallucinations.

### Decision 3.6: Worker Fencing & Zombie Protection
- **Classification**: **[Design Decision Grounded in Staging]**
- **Rationale**: Long-running background TTS jobs may exceed lease duration. If a worker gets partitioned or GC-stalled, a subsequent worker may claim the job. To prevent zombie workers from overwriting new audio or corrupting database state, each job claim issues a monotonic `lease_token`. Updates require `lease_token` matching.


---

## Section 4: Target Schema Architecture & Relationships

The target schema introduces four dedicated tables and alters `public.pages` strictly with backward-compatible defaults.

### 4.1 Schema Migration DDL (Target State)
```sql
-- 1. Alter pages to track story source invalidation tokens
ALTER TABLE public.pages
  ADD COLUMN IF NOT EXISTS story_source_change_token text NOT NULL DEFAULT gen_random_uuid();

-- 2. Story Segments (Canonical cross-language semantic scenes)
CREATE TABLE IF NOT EXISTS public.story_segments (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  segment_order integer NOT NULL,
  title text NOT NULL DEFAULT '',
  start_page_number integer NOT NULL,
  end_page_number integer NOT NULL,
  source_page_ids uuid[] NOT NULL,
  source_tokens_hash text NOT NULL,
  status text NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'ready', 'stale', 'archived')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_story_segments_order UNIQUE (book_id, segment_order)
);

-- 3. Localized Scripts & Audio Renditions
CREATE TABLE IF NOT EXISTS public.story_segment_localizations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  segment_id uuid NOT NULL REFERENCES public.story_segments(id) ON DELETE CASCADE,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  language text NOT NULL CHECK (language IN ('ar', 'pl', 'en')),
  narrator_persona text NOT NULL DEFAULT 'shahrazad_egyptian',
  script_text text NOT NULL,
  script_summary text NOT NULL DEFAULT '',
  audio_storage_path text,
  audio_duration_seconds numeric(8, 2) DEFAULT 0.00,
  audio_format text DEFAULT 'audio/mpeg',
  audio_status text NOT NULL DEFAULT 'pending' CHECK (audio_status IN ('pending', 'processing', 'ready', 'failed')),
  fts_doc tsvector,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_segment_localization UNIQUE (segment_id, language)
);

-- 4. Story Listening Sessions & Checkpoint Tracking
CREATE TABLE IF NOT EXISTS public.story_listening_sessions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  current_segment_id uuid REFERENCES public.story_segments(id) ON DELETE SET NULL,
  language text NOT NULL DEFAULT 'ar',
  last_position_seconds numeric(8, 2) NOT NULL DEFAULT 0.00,
  last_checkpoint_at timestamptz NOT NULL DEFAULT now(),
  session_status text NOT NULL DEFAULT 'active' CHECK (session_status IN ('active', 'paused', 'completed')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- 5. User Segment Completions (Source for No-Spoiler RAG)
CREATE TABLE IF NOT EXISTS public.story_completions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  segment_id uuid NOT NULL REFERENCES public.story_segments(id) ON DELETE CASCADE,
  language text NOT NULL,
  source_tokens_hash text NOT NULL,
  completed_at timestamptz NOT NULL DEFAULT now(),
  total_playback_seconds numeric(8, 2) NOT NULL DEFAULT 0.00,
  CONSTRAINT uq_user_segment_completion UNIQUE (user_id, segment_id, language)
);

-- 6. Background Jobs Queue (PostgreSQL transactional queue)
CREATE TABLE IF NOT EXISTS public.story_jobs_queue (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  segment_id uuid REFERENCES public.story_segments(id) ON DELETE CASCADE,
  job_type text NOT NULL CHECK (job_type IN ('segment_analysis', 'script_generation', 'tts_synthesis', 'audio_cleanup')),
  target_language text NOT NULL DEFAULT 'ar',
  payload jsonb NOT NULL DEFAULT '{}'::jsonb,
  status text NOT NULL DEFAULT 'queued' CHECK (status IN ('queued', 'running', 'completed', 'failed', 'dead_letter')),
  attempts integer NOT NULL DEFAULT 0,
  max_attempts integer NOT NULL DEFAULT 3,
  lease_token uuid,
  leased_until timestamptz,
  last_error text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- 7. Storage Cleanup Queue
CREATE TABLE IF NOT EXISTS public.story_storage_cleanup_queue (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid REFERENCES public.books(id) ON DELETE CASCADE,
  storage_path text NOT NULL,
  cleanup_mode text NOT NULL DEFAULT 'single_object' CHECK (cleanup_mode IN ('single_object', 'prefix_directory')),
  status text NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'completed', 'failed')),
  attempts integer NOT NULL DEFAULT 0,
  error_message text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
```


---

## Section 5: Playback Verification, Server Checkpoints & Anti-Spoofing

To eliminate client spoofing and prevent marking segments completed through simulated client payload submissions, completion is governed by strict server-side validation.

### 5.1 Validation Protocol
1. **Periodic Server Checkpoints (`story_record_playback_checkpoint`)**:
   - The audio player submits checkpoints every 5–10 seconds.
   - The RPC verifies that `elapsed_real_seconds = now() - last_checkpoint_at`.
   - The submitted `position_seconds` must satisfy:
     `position_seconds <= last_position_seconds + elapsed_real_seconds + jitter_allowance (e.g. 3.0s)`.
   - If position advances faster than physical real-time playback permits, the checkpoint is flagged and rejected.
   - `last_position_seconds` and `last_checkpoint_at` are updated transactionally.

2. **Completion Verification (`story_complete_playback`)**:
   - Requires an active listening session matching `(session_id, book_id, segment_id, language)`.
   - Verifies that audio rendition exists and has status `'ready'`.
   - The server verifies:
     `position_seconds >= (localization.audio_duration_seconds * 0.90)` (at least 90% listened).
   - Verifies that `last_position_seconds` stored on the session is already close to the finish point (preventing a single jump from 0s to end).
   - Verifies that the segment's `source_tokens_hash` matches current source pages. If pages were modified mid-playback, completion is rejected or flagged as stale.
   - Inserts into `public.story_completions` using `ON CONFLICT (user_id, segment_id, language) DO UPDATE`.


---

## Section 6: No-Spoilers Semantic RAG & Multilingual Full-Text Search

### 6.1 Strict Anti-Spoiler Context Boundary
When a child asks Shahrazad a question (e.g., *"مين ده؟"* or *"هو إيه اللي هيحصل لبوليك؟"*), the retrieval engine must never fetch future narrative scenes.

- **Completed Segments Filter**:
  The retrieval query joins `story_segment_localizations` against `story_completions` matching `auth.uid()`:
  ```sql
  WHERE comp.user_id = auth.uid()
    AND comp.book_id = p_book_id
    AND comp.source_tokens_hash = seg.source_tokens_hash
    AND loc.language = p_language
  ```
- If the story source pages were updated after completion, `source_tokens_hash` mismatch prevents stale or altered text from serving as verified ground truth.

### 6.2 Multilingual FTS Vector Implementation
Because Staging lacks Polish dictionary support in `pg_catalog`:
```sql
CREATE OR REPLACE FUNCTION public.compute_story_fts_doc(p_language text, p_text text, p_summary text)
RETURNS tsvector AS $$
BEGIN
  IF p_language = 'ar' THEN
    RETURN setweight(to_tsvector('arabic', coalesce(p_summary, '')), 'A') ||
           setweight(to_tsvector('arabic', coalesce(p_text, '')), 'B');
  ELSIF p_language = 'en' THEN
    RETURN setweight(to_tsvector('english', coalesce(p_summary, '')), 'A') ||
           setweight(to_tsvector('english', coalesce(p_text, '')), 'B');
  ELSE
    -- Polish and fallback: use 'simple' dictionary (case-folded, punctuation-stripped)
    RETURN setweight(to_tsvector('simple', coalesce(p_summary, '')), 'A') ||
           setweight(to_tsvector('simple', coalesce(p_text, '')), 'B');
  END IF;
END;
$$ LANGUAGE plpgsql IMMUTABLE;
```


---

## Section 7: Worker Lifecycle, Lease Fencing & Storage Cleanup

### 7.1 Queue Protocol: Claim, Renew, Finish, Retry
1. **Claiming with Monotonic Lease Token (`claim_story_job`)**:
   - Uses `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1`.
   - Generates a fresh `v_lease_token := gen_random_uuid()`.
   - Sets `leased_until := now() + interval '5 minutes'`.
   - Returns the job payload along with `lease_token`.

2. **Fencing Execution**:
   - Every worker mutation (saving script, uploading audio path, updating status) requires supplying `job_id` and `lease_token`.
   - If `leased_until < now()` or `lease_token != current_lease_token`, the write fails immediately.

3. **Lease Renewal (`heartbeat_story_job`)**:
   - Long TTS synthesis calls periodically ping `heartbeat_story_job(p_job_id, p_lease_token)` to extend `leased_until` by 3 minutes.

4. **Failure & Dead-Letter Handling**:
   - If `attempts >= max_attempts`, mark job as `'dead_letter'` and alert diagnostics.

### 7.2 Two-Stage Storage Cleanup Queue
When books or pages are modified or deleted, audio files must not be orphaned in Supabase Storage.
1. **Single Object Cleanup**:
   When re-synthesizing a segment audio, old audio storage path is enqueued in `story_storage_cleanup_queue` with mode `'single_object'`.
2. **Prefix Directory Cleanup**:
   When an entire book is deleted (`ON DELETE CASCADE`), a database trigger or cleanup RPC registers `books/{book_id}/` in `story_storage_cleanup_queue` with mode `'prefix_directory'`.
3. **Worker Processing**:
   A lightweight cleanup worker processes the cleanup queue, calling `supabase.storage.from('story-audio').remove([...])` safely.


---

## Section 8: Security, RLS & Privilege Remediation

### 8.1 Empirical Security Defect Uncovered by Audit
The audit revealed that both `anon` and `authenticated` roles possess full table grants on public tables:
- `books`: `DELETE, INSERT, REFERENCES, SELECT, TRIGGER, TRUNCATE, UPDATE`
- `pages`: `DELETE, INSERT, REFERENCES, SELECT, TRIGGER, TRUNCATE, UPDATE`
- `difficult_words`, `reading_sessions`, `review_attempts`, `settings`, etc.: Identical full grants.

**Critical Threat Analysis**:
While RLS restricts `SELECT, INSERT, UPDATE, DELETE` based on row conditions, **RLS does not restrict `TRUNCATE`**. A client with table-level TRUNCATE privilege could theoretically execute truncate operations if exposed. Furthermore, `TRIGGER` and `REFERENCES` privileges should never be granted to public roles.

### 8.2 Remediation Policy (Safe Migration Step)
```sql
-- Revoke destructive and administrative privileges from anon and authenticated
REVOKE TRUNCATE, TRIGGER, REFERENCES ON ALL TABLES IN SCHEMA public FROM anon, authenticated;

-- Ensure anon has zero access to private educational tables
REVOKE ALL ON TABLE public.books, public.pages, public.reading_sessions,
                     public.reading_attempts, public.review_attempts,
                     public.difficult_words, public.lesson_events,
                     public.settings, public.profiles FROM anon;

-- Protect story completions: Authenticated users CANNOT directly INSERT or UPDATE completions
REVOKE INSERT, UPDATE, DELETE, TRUNCATE ON public.story_completions FROM authenticated;
GRANT SELECT ON public.story_completions TO authenticated;
-- Insertions/updates to completions are executed solely through SECURITY DEFINER RPC story_complete_playback
```

### 8.3 Storage RLS Policies for `story-audio`
```sql
-- Bucket is private: public = false
-- Storage objects policy: authenticated users can only READ audio belonging to books they own
CREATE POLICY "Users can stream audio for own books"
ON storage.objects FOR SELECT TO authenticated
USING (
  bucket_id = 'story-audio' AND
  EXISTS (
    SELECT 1 FROM public.books b
    WHERE b.id::text = (storage.foldername(name))[2]
      AND b.user_id = auth.uid()
  )
);
```


---

## Section 9: Guarded, Idempotent Staging Migration Script

The following complete migration script is designed to run safely on Staging. It incorporates strict pre-checks, idempotency guards, and verified backward compatibility.

```sql
-- ============================================================================
-- SHAHRAZAD SUBSYSTEM MIGRATION (STAGING GUARDED)
-- Target Engine: PostgreSQL 17.6
-- Pre-checked against Shahrazad_Staging_ReadOnly_Audit.csv
-- ============================================================================

BEGIN;

-- 1. Security Pre-Remediation: Revoke dangerous grants on existing tables
REVOKE TRUNCATE, TRIGGER, REFERENCES ON ALL TABLES IN SCHEMA public FROM anon, authenticated;

-- 2. Alter pages: Add story source tracking token safely
DO $$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM information_schema.columns 
    WHERE table_schema = 'public' AND table_name = 'pages' AND column_name = 'story_source_change_token'
  ) THEN
    ALTER TABLE public.pages ADD COLUMN story_source_change_token text NOT NULL DEFAULT gen_random_uuid()::text;
  END IF;
END $$;

-- 3. Trigger to update story_source_change_token on page edit
CREATE OR REPLACE FUNCTION public.handle_page_source_change()
RETURNS trigger AS $$
BEGIN
  IF (OLD.ocr_text IS DISTINCT FROM NEW.ocr_text) OR 
     (OLD.lesson IS DISTINCT FROM NEW.lesson) OR
     (OLD.page_number IS DISTINCT FROM NEW.page_number) THEN
    NEW.story_source_change_token := gen_random_uuid()::text;
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS t_page_source_token_update ON public.pages;
CREATE TRIGGER t_page_source_token_update
BEFORE UPDATE ON public.pages
FOR EACH ROW
EXECUTE FUNCTION public.handle_page_source_change();

-- 4. Story Segments
CREATE TABLE IF NOT EXISTS public.story_segments (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  segment_order integer NOT NULL,
  title text NOT NULL DEFAULT '',
  start_page_number integer NOT NULL,
  end_page_number integer NOT NULL,
  source_page_ids uuid[] NOT NULL,
  source_tokens_hash text NOT NULL,
  status text NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'ready', 'stale', 'archived')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_story_segments_order UNIQUE (book_id, segment_order)
);

-- 5. Localizations
CREATE TABLE IF NOT EXISTS public.story_segment_localizations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  segment_id uuid NOT NULL REFERENCES public.story_segments(id) ON DELETE CASCADE,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  language text NOT NULL CHECK (language IN ('ar', 'pl', 'en')),
  narrator_persona text NOT NULL DEFAULT 'shahrazad_egyptian',
  script_text text NOT NULL,
  script_summary text NOT NULL DEFAULT '',
  audio_storage_path text,
  audio_duration_seconds numeric(8, 2) DEFAULT 0.00,
  audio_format text DEFAULT 'audio/mpeg',
  audio_status text NOT NULL DEFAULT 'pending' CHECK (audio_status IN ('pending', 'processing', 'ready', 'failed')),
  fts_doc tsvector,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),
  CONSTRAINT uq_segment_localization UNIQUE (segment_id, language)
);

-- 6. Listening Sessions
CREATE TABLE IF NOT EXISTS public.story_listening_sessions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  current_segment_id uuid REFERENCES public.story_segments(id) ON DELETE SET NULL,
  language text NOT NULL DEFAULT 'ar',
  last_position_seconds numeric(8, 2) NOT NULL DEFAULT 0.00,
  last_checkpoint_at timestamptz NOT NULL DEFAULT now(),
  session_status text NOT NULL DEFAULT 'active' CHECK (session_status IN ('active', 'paused', 'completed')),
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- 7. User Segment Completions
CREATE TABLE IF NOT EXISTS public.story_completions (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  segment_id uuid NOT NULL REFERENCES public.story_segments(id) ON DELETE CASCADE,
  language text NOT NULL,
  source_tokens_hash text NOT NULL,
  completed_at timestamptz NOT NULL DEFAULT now(),
  total_playback_seconds numeric(8, 2) NOT NULL DEFAULT 0.00,
  CONSTRAINT uq_user_segment_completion UNIQUE (user_id, segment_id, language)
);

-- 8. Jobs Queue
CREATE TABLE IF NOT EXISTS public.story_jobs_queue (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid NOT NULL REFERENCES public.books(id) ON DELETE CASCADE,
  segment_id uuid REFERENCES public.story_segments(id) ON DELETE CASCADE,
  job_type text NOT NULL CHECK (job_type IN ('segment_analysis', 'script_generation', 'tts_synthesis', 'audio_cleanup')),
  target_language text NOT NULL DEFAULT 'ar',
  payload jsonb NOT NULL DEFAULT '{}'::jsonb,
  status text NOT NULL DEFAULT 'queued' CHECK (status IN ('queued', 'running', 'completed', 'failed', 'dead_letter')),
  attempts integer NOT NULL DEFAULT 0,
  max_attempts integer NOT NULL DEFAULT 3,
  lease_token uuid,
  leased_until timestamptz,
  last_error text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- 9. Storage Cleanup Queue
CREATE TABLE IF NOT EXISTS public.story_storage_cleanup_queue (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  book_id uuid REFERENCES public.books(id) ON DELETE CASCADE,
  storage_path text NOT NULL,
  cleanup_mode text NOT NULL DEFAULT 'single_object' CHECK (cleanup_mode IN ('single_object', 'prefix_directory')),
  status text NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'completed', 'failed')),
  attempts integer NOT NULL DEFAULT 0,
  error_message text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);

-- 10. Enable RLS on all newly created tables
ALTER TABLE public.story_segments ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_segment_localizations ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_listening_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_completions ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_jobs_queue ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.story_storage_cleanup_queue ENABLE ROW LEVEL SECURITY;

-- 11. Policies for Story Tables
CREATE POLICY "own story segments" ON public.story_segments
  FOR ALL TO authenticated
  USING (EXISTS (SELECT 1 FROM public.books b WHERE b.id = story_segments.book_id AND b.user_id = auth.uid()))
  WITH CHECK (EXISTS (SELECT 1 FROM public.books b WHERE b.id = story_segments.book_id AND b.user_id = auth.uid()));

CREATE POLICY "own story localizations" ON public.story_segment_localizations
  FOR ALL TO authenticated
  USING (EXISTS (SELECT 1 FROM public.books b WHERE b.id = story_segment_localizations.book_id AND b.user_id = auth.uid()))
  WITH CHECK (EXISTS (SELECT 1 FROM public.books b WHERE b.id = story_segment_localizations.book_id AND b.user_id = auth.uid()));

CREATE POLICY "own listening sessions" ON public.story_listening_sessions
  FOR ALL TO authenticated
  USING (user_id = auth.uid())
  WITH CHECK (user_id = auth.uid());

CREATE POLICY "read own completions" ON public.story_completions
  FOR SELECT TO authenticated
  USING (user_id = auth.uid());

-- Completions cannot be inserted/updated directly by authenticated clients
REVOKE INSERT, UPDATE, DELETE ON public.story_completions FROM authenticated;

-- Service role policies for worker operations
CREATE POLICY "service role full jobs" ON public.story_jobs_queue
  FOR ALL TO service_role USING (true) WITH CHECK (true);

CREATE POLICY "service role full cleanup" ON public.story_storage_cleanup_queue
  FOR ALL TO service_role USING (true) WITH CHECK (true);

-- 12. Create Functions for Checkpointing, Completion & Search
CREATE OR REPLACE FUNCTION public.story_record_playback_checkpoint(
  p_session_id uuid,
  p_segment_id uuid,
  p_position_seconds numeric
)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_session public.story_listening_sessions%ROWTYPE;
  v_elapsed_seconds numeric;
  v_allowed_delta numeric;
BEGIN
  IF auth.uid() IS NULL THEN
    RAISE EXCEPTION 'Authentication required';
  END IF;

  SELECT * INTO v_session
  FROM public.story_listening_sessions
  WHERE id = p_session_id AND user_id = auth.uid()
  FOR UPDATE;

  IF NOT FOUND THEN
    RAISE EXCEPTION 'Active session not found';
  END IF;

  -- Validate monotonic advancement against real elapsed server time
  v_elapsed_seconds := EXTRACT(EPOCH FROM (now() - v_session.last_checkpoint_at));
  v_allowed_delta := v_elapsed_seconds + 3.0; -- 3 seconds network/buffer jitter allowance

  IF (p_position_seconds > v_session.last_position_seconds + v_allowed_delta) THEN
    -- Client jumped forward faster than physical time elapsed; clamp or log
    p_position_seconds := v_session.last_position_seconds + v_allowed_delta;
  END IF;

  UPDATE public.story_listening_sessions
  SET current_segment_id = p_segment_id,
      last_position_seconds = p_position_seconds,
      last_checkpoint_at = now(),
      updated_at = now()
  WHERE id = p_session_id;

  RETURN jsonb_build_object('status', 'ok', 'recorded_position', p_position_seconds);
END;
$$;

CREATE OR REPLACE FUNCTION public.story_complete_playback(
  p_session_id uuid,
  p_segment_id uuid,
  p_position_seconds numeric
)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_session public.story_listening_sessions%ROWTYPE;
  v_seg public.story_segments%ROWTYPE;
  v_loc public.story_segment_localizations%ROWTYPE;
BEGIN
  IF auth.uid() IS NULL THEN
    RAISE EXCEPTION 'Authentication required';
  END IF;

  SELECT * INTO v_session
  FROM public.story_listening_sessions
  WHERE id = p_session_id AND user_id = auth.uid();

  IF NOT FOUND THEN
    RAISE EXCEPTION 'Session not found';
  END IF;

  SELECT * INTO v_seg
  FROM public.story_segments
  WHERE id = p_segment_id AND book_id = v_session.book_id;

  IF NOT FOUND THEN
    RAISE EXCEPTION 'Segment not found';
  END IF;

  SELECT * INTO v_loc
  FROM public.story_segment_localizations
  WHERE segment_id = p_segment_id AND language = v_session.language;

  IF NOT FOUND OR v_loc.audio_status != 'ready' THEN
    RAISE EXCEPTION 'Segment audio localization is not ready';
  END IF;

  -- Require having reached at least 90% of playback duration
  IF p_position_seconds < (v_loc.audio_duration_seconds * 0.90) AND v_loc.audio_duration_seconds > 5.0 THEN
    RAISE EXCEPTION 'Playback incomplete: position % under threshold %', p_position_seconds, (v_loc.audio_duration_seconds * 0.90);
  END IF;

  -- Insert or update completion record
  INSERT INTO public.story_completions (
    user_id, book_id, segment_id, language, source_tokens_hash, total_playback_seconds
  ) VALUES (
    auth.uid(), v_session.book_id, p_segment_id, v_session.language, v_seg.source_tokens_hash, p_position_seconds
  )
  ON CONFLICT (user_id, segment_id, language) DO UPDATE
  SET total_playback_seconds = EXCLUDED.total_playback_seconds,
      completed_at = now();

  RETURN jsonb_build_object('status', 'completed', 'segment_id', p_segment_id);
END;
$$;

COMMIT;
```


---

## Section 10: Staging Acceptance Test Suite & Verification Criteria

The following test suite must be executed on Staging post-migration before any application integration. Each test specifies explicit PASS / FAIL criteria.

| Test ID | Objective | Test Execution Procedure | Expected Behavior | Pass / Fail Criteria |
| :--- | :--- | :--- | :--- | :--- |
| **TEST-01** | **Table & Column Integrity** | Query `information_schema.tables` and `columns` for all new `story_*` entities and `pages.story_source_change_token`. | All 6 new tables present; column types match target DDL. | **PASS**: 6 tables and token column present. <br>**FAIL**: Missing table/column or constraint discrepancy. |
| **TEST-02** | **Privilege Confinement (TRUNCATE & Grants)** | Attempt `TRUNCATE TABLE public.books` and `TRUNCATE TABLE public.story_completions` using `authenticated` and `anon` roles. | PostgreSQL rejects operation with `permission denied`. | **PASS**: Error `42501: permission denied for table`. <br>**FAIL**: Any table truncated or permission allowed. |
| **TEST-03** | **Direct Completion Insertion Block** | Execute `INSERT INTO public.story_completions (...)` directly as authenticated user (not via RPC). | Direct write is denied by table grant revoking. | **PASS**: Error `permission denied for table story_completions`. <br>**FAIL**: Direct row insert succeeds. |
| **TEST-04** | **Page Source Token Auto-Invalidation** | Update `ocr_text` or `lesson` on a test page in `pages`. Compare `story_source_change_token` before and after. | Trigger `t_page_source_token_update` generates fresh UUID. | **PASS**: New token differs from previous token. <br>**FAIL**: Token remains unchanged after update. |
| **TEST-05** | **Anti-Spoofing Playback Checkpoint** | Create session, record checkpoint at 0s, immediately call checkpoint with 100s position (0 elapsed time). | Server calculates `allowed_delta` and clamps position. | **PASS**: Recorded position clamped to ~3s. <br>**FAIL**: 100s accepted without real elapsed time. |
| **TEST-06** | **Completion Threshold Enforcement** | Invoke `story_complete_playback` with position = 10s on a 60s segment. | RPC raises exception `Playback incomplete`. | **PASS**: Exception raised; no completion recorded. <br>**FAIL**: Completion inserted prematurely. |
| **TEST-07** | **No-Spoilers RAG Filtering** | Insert segment 1 (completed) and segment 2 (uncompleted). Search RAG context as child user. | RAG query returns only segment 1 content. | **PASS**: Zero leakage of segment 2 content. <br>**FAIL**: Uncompleted segment 2 content returned. |
| **TEST-08** | **Polish FTS Compatibility** | Execute `compute_story_fts_doc('pl', 'Tekst o smoku', 'Smok wawelski')`. | Function executes using `simple` dictionary without runtime error. | **PASS**: tsvector generated; zero dictionary errors. <br>**FAIL**: Exception `text search configuration "polish" does not exist`. |
| **TEST-09** | **Worker Lease Fencing** | Worker A claims job (lease token A). Worker B artificially steals or updates lease. Worker A attempts completion with token A. | Worker A's update is rejected due to token mismatch. | **PASS**: Zero overwrites by expired worker. <br>**FAIL**: Stale worker overwrites state. |
| **TEST-10** | **Book Deletion Cleanup Cascade** | Delete a test book with registered audio paths. Query `story_storage_cleanup_queue`. | Prefix or object cleanup row is registered; cascade cleans DB records. | **PASS**: Storage paths enqueued for asynchronous file deletion. <br>**FAIL**: Dangling storage paths without cleanup entry. |


---

## Section 11: End-to-End Implementation Roadmap

Following verification of the acceptance tests on Staging:

```
[Phase 1: Database Migration on Staging]
  ├── Execute Guarded Staging Migration Script
  ├── Run Acceptance Tests TEST-01 through TEST-10
  └── Verify Storage Bucket 'story-audio' configuration

[Phase 2: Backend Job Engine & TTS Integration]
  ├── Implement Background Job Worker (TypeScript / Node runtime on Cloudflare)
  ├── Hook Gemini Script Generation (Egyptian Arabic / Polish / English personas)
  ├── Connect TTS synthesis pipeline with lease-heartbeat mechanism
  └── Connect Storage Cleanup Worker

[Phase 3: Client Experience & Interactive Storytelling]
  ├── Build Book Story Route (/books/:bookId/story)
  ├── Connect Audio Streaming Player with Server Checkpointing
  ├── Connect Push-to-Talk Question Drawer (No-Spoiler RAG)
  └── Implement Story Invalidation Banner when OCR is modified

[Phase 4: Field Testing & Production Readiness]
  ├── Real-device audio latency & child interruption QA
  ├── Parent Dashboard story completion metrics
  └── Production Readiness Sign-Off
```
