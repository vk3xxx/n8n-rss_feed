# RSS Post Flow with Deduplication

This n8n workflow automatically posts RSS articles to multiple social media platforms while preventing duplicate posts.

## How Deduplication Works

### 1. **Store Node** - Main Deduplication Logic
- **Memory Storage**: Uses n8n's global workflow static data to persist state across executions
- **30-Day TTL**: Articles are remembered as "posted" for 30 days to prevent reposting
- **Smart Key Generation**: Creates unique identifiers using:
  - Primary: Normalized URL (strips tracking parameters, fragments, etc.)
  - Fallback: RSS GUID if available
  - Fallback: Hash of title + date + source

### 2. **Deduplication Process**
- **URL Normalization**: Removes tracking parameters (utm_*, gclid, fbclid, etc.)
- **Case Insensitive**: Hostnames are normalized to lowercase
- **Fragment Removal**: URL hashes are ignored
- **Trailing Slash**: Removes trailing slashes for consistency

### 3. **State Management**
- **Posted Articles**: `state.posted[key] = timestamp` (30-day retention)
- **Pending Articles**: `state.pending[key] = timestamp` (2-hour reservation)
- **Automatic Cleanup**: Expired entries are removed automatically
- **Memory Limits**: Maximum 5000 keys to prevent memory bloat

### 4. **Flow Protection**
- **Race Condition Prevention**: Pending state prevents multiple executions from processing the same article
- **Crash Recovery**: Stale pending entries are cleaned up after 2 hours
- **Batch Processing**: Articles are processed in batches of 3 to avoid overwhelming APIs

### 5. **Debug Logging**
- **Console Output**: Shows deduplication statistics in n8n execution logs
- **Article Tracking**: Lists which articles are being processed vs. skipped
- **State Monitoring**: Displays current counts of posted and pending articles

## Flow Architecture

```
RSS Feeds → Store (Deduplication) → Debug Log → Merge + If → LLM Chain → Data Transform → Social Media
```

**Key Components:**
- **Store**: Handles deduplication and prevents reposts
- **Debug Log**: Shows processing statistics and article details
- **LLM Chain**: Generates social media content using AI
- **Data Transform**: Ensures data structure consistency and prevents errors
- **Social Media**: Posts to Mastodon, Bluesky, Discord, and Telegram

## Configuration

- **TTL**: 30 days (configurable in Store node)
- **Pending TTL**: 2 hours (configurable in Store node)
- **Max Keys**: 5000 (configurable in Store node)
- **Batch Size**: 3 articles per batch

## Monitoring

Check the n8n execution logs to see deduplication statistics:
- Number of new articles processed
- Number of articles skipped (already posted)
- Total articles in memory
- Current pending articles

## Benefits

1. **No Duplicate Posts**: Articles are never posted twice within 30 days
2. **Memory Efficient**: Automatic cleanup prevents memory bloat
3. **Crash Safe**: Handles workflow interruptions gracefully
4. **Multi-Source**: Works across multiple RSS feeds
5. **Performance**: Fast key generation and lookup
6. **Debugging**: Clear logging for troubleshooting

## Technical Details

- Uses `$getWorkflowStaticData('global')` for persistent storage
- SHA1 hashing for fallback key generation
- URL normalization using native JavaScript URL API
- Automatic garbage collection with configurable TTLs
- Thread-safe pending state management
