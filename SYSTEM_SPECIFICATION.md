# RSS Feed Processing System Specification

## System Overview

You are tasked with recreating an n8n workflow that processes RSS feeds and automatically posts content to multiple social media platforms while preventing duplicate posts. This system implements enterprise-grade deduplication, intelligent content generation, and multi-platform distribution.

## Core Design Philosophy

### 1. **Deduplication-First Architecture**
- **Primary Goal**: Never post the same article twice within a 30-day period
- **Memory Efficiency**: Automatic cleanup prevents memory bloat
- **Crash Recovery**: Handles workflow interruptions gracefully
- **Race Condition Prevention**: Prevents multiple executions from processing the same article

### 2. **Data Integrity & Validation**
- **Structure Validation**: Ensure data consistency before API calls
- **Fallback Handling**: Provide default values for missing fields
- **Error Isolation**: Individual node failures don't crash the entire workflow
- **Logging & Monitoring**: Comprehensive visibility into system operation

### 3. **Scalable RSS Processing**
- **Multi-Source Support**: Handle multiple RSS feeds simultaneously
- **Batch Processing**: Process articles in controlled batches to avoid API rate limits
- **Intelligent Scheduling**: Different trigger intervals for different feed sources

## System Architecture

### Flow Structure
```
RSS Feeds (Multiple Sources) 
    ↓
Store Node (Deduplication Engine)
    ↓
Debug Log (Processing Statistics)
    ↓
Merge + If (Data Validation & URL Check)
    ↓
TinyURL Creation (Link Shortening)
    ↓
LLM Chain (AI Content Generation)
    ↓
Data Transform (Structure Validation)
    ↓
Social Media Distribution (4 Platforms)
    ↓
Commit Node (Success Confirmation)
    ↓
Wait Nodes (Rate Limiting)
```

### Node Specifications

#### 1. **RSS Feed Sources**
- **AR Newsline**: `https://www.arnewsline.org/?format=rss`
- **NASA Aeronautics**: `https://www.nasa.gov/aeronautics/feed/`
- **Fox US Politics**: `https://moxie.foxnews.com/google-publisher/politics.xml`
- **Fox World**: `https://moxie.foxnews.com/google-publisher/world.xml`

#### 2. **Store Node (Deduplication Engine)**
```javascript
// Core deduplication logic
const state = $getWorkflowStaticData('global');
state.posted = state.posted || {};   // { key: timestamp }
state.pending = state.pending || {}; // { key: timestamp }

const NOW = Date.now();
const TTL_MS = 30*24*60*60*1000;     // 30 days
const PENDING_TTL = 2*60*60*1000;    // 2 hours
const MAX_KEYS = 5000;
const MAX_PER_RUN = 0;               // 0 = unlimited

// URL normalization function
function normalise(u) {
  try {
    const url = new URL(String(u));
    url.hash = '';                               // ignore fragments
    url.hostname = url.hostname.toLowerCase();   // host is case-insensitive
    url.pathname = url.pathname.replace(/\/+$/, ''); // trim trailing slash
    // drop tracking parameters
    for (const p of [
      'utm_source','utm_medium','utm_campaign','utm_term','utm_content',
      'gclid','fbclid','mc_cid','mc_eid','igshid','ref','campaign_id'
    ]) url.searchParams.delete(p);
    return url.toString();
  } catch {
    return (u || '').toString();
  }
}

// Key generation with fallbacks
function keyForItem(j) {
  const link = (typeof j.link === 'string' && j.link) ? normalise(j.link) : '';
  if (link) return `u:${link}`;
  if (j.guid) return `g:${String(j.guid)}`;
  const raw = `${j.title || ''}::${j.pubDate || j.isoDate || ''}::${j.source || ''}`;
  const crypto = require('crypto');
  return 'h:' + crypto.createHash('sha1').update(raw).digest('hex');
}

// Main processing loop
for (const item of items) {
  const j = item.json || {};
  const key = keyForItem(j);
  if (!key) continue;

  // Check if already posted recently
  if (state.posted[key] && (NOW - state.posted[key]) < TTL_MS) {
    continue;
  }

  // Check if already in-flight
  if (state.pending[key] && (NOW - state.pending[key]) < PENDING_TTL) {
    continue;
  }

  // Reserve for processing
  state.pending[key] = NOW;
  out.push({ json: { ...j, dedupe_key: key } });
}
```

#### 3. **Debug Log Node**
```javascript
// Debug logging for deduplication
const items = await $input.all();

if (items.length === 0) {
  console.log('No new articles to process - all were deduplicated');
} else {
  console.log(`Processing ${items.length} new articles:`);
  for (const item of items) {
    const j = item.json || {};
    console.log(`- ${j.title || 'No title'} (${j.dedupe_key})`);
  }
}

return items;
```

#### 4. **Data Transform Node**
```javascript
// Data transformation and validation for social media nodes
const items = await $input.all();
const out = [];

for (const item of items) {
  try {
    // Ensure we have the required structure
    const text = item.json?.text || item.json?.content || 'No content available';
    
    // Create a clean, validated data structure
    const cleanItem = {
      json: {
        text: text,
        title: item.json?.title || 'RSS Article',
        link: item.json?.link || '',
        dedupe_key: item.json?.dedupe_key || '',
        content: text,
        ...item.json
      }
    };
    
    out.push(cleanItem);
  } catch (error) {
    console.error('Error processing item:', error, item);
    continue;
  }
}

return out;
```

#### 5. **LLM Chain Node**
- **Model**: OpenRouter Chat Model
- **Prompt**: "Please summarise into social media post, combine with the short URL and make sure it makes sense. Maximum characters 200. Go!! {{ $json.content }} {{ $('Tiny').item.json.data.tiny_url }}"
- **Output**: Structured text for social media platforms

#### 6. **Social Media Distribution Nodes**

##### Mastodon Node
- **Type**: `n8n-nodes-mastodon.mastodon`
- **URL**: `https://masto.supes.com`
- **Text**: `{{ $json.text || $json.content || 'RSS Article Update' }}`
- **Error Handling**: `onError: continueRegularOutput`

##### Bluesky Node
- **Type**: `@muench-dev/n8n-nodes-bluesky.bluesky`
- **Text**: `{{ $json.text || $json.content || 'RSS Article Update' }}`
- **Error Handling**: `onError: continueRegularOutput`

##### Discord Node
- **Type**: `n8n-nodes-base.discord`
- **Authentication**: Webhook
- **Content**: `{{ $json.text || $json.content || 'RSS Article Update' }}`
- **Error Handling**: `onError: continueRegularOutput`

##### Telegram Node
- **Type**: `n8n-nodes-base.telegram`
- **Chat ID**: `-1002899637373`
- **Text**: `{{ $json.text || $json.content || 'RSS Article Update' }}`
- **Error Handling**: `onError: continueRegularOutput`

#### 7. **Commit Node**
```javascript
// Commit success (promote pending → posted)
const state = $getWorkflowStaticData('global');
state.posted = state.posted || {};
state.pending = state.pending || {};

const NOW = Date.now();
const TTL_MS = 30*24*60*60*1000;

const items = await $input.all();
const committed = new Set();

for (const item of items) {
  const j = item.json || {};
  const key = j.dedupe_key;
  if (!key || committed.has(key)) continue;

  state.posted[key] = NOW;   // promote
  delete state.pending[key];  // clear reservation
  committed.add(key);
}

// TTL cleanup
for (const k of Object.keys(state.posted)) {
  if (NOW - state.posted[k] >= TTL_MS) delete state.posted[k];
}

return items;
```

#### 8. **Wait Nodes (Rate Limiting)**
- **Wait Short**: 36 minutes
- **Wait Long**: 120 minutes
- **Purpose**: Prevent API rate limiting and provide controlled processing intervals

#### 9. **Trigger Nodes (Scheduling)**
- **Trigger1**: 4-minute intervals (Fox World)
- **Trigger2**: 6-minute intervals (Fox US Politics)
- **Trigger3**: 8-minute intervals (RSS NASA)
- **Trigger4**: 10-minute intervals (AR Newsline)

## Implementation Requirements

### 1. **Node Dependencies**
- `n8n-nodes-base.rssFeedRead` - RSS feed reading
- `@n8n/n8n-nodes-langchain.chainLlm` - AI content generation
- `@n8n/n8n-nodes-langchain.lmChatOpenRouter` - OpenRouter integration
- `@muench-dev/n8n-nodes-bluesky.bluesky` - Bluesky posting
- `n8n-nodes-mastodon.mastodon` - Mastodon posting
- `n8n-nodes-base.discord` - Discord webhook
- `n8n-nodes-base.telegram` - Telegram posting
- `n8n-nodes-base.httpRequest` - TinyURL API calls
- `n8n-nodes-base.code` - Custom JavaScript logic
- `n8n-nodes-base.merge` - Data combination
- `n8n-nodes-base.if` - Conditional logic
- `n8n-nodes-base.splitInBatches` - Batch processing
- `n8n-nodes-base.wait` - Rate limiting
- `n8n-nodes-base.scheduleTrigger` - Automated execution

### 2. **Credential Requirements**
- **OpenRouter API**: For AI content generation
- **TinyURL API**: For link shortening
- **Mastodon OAuth2**: For posting to Mastodon
- **Bluesky API**: For posting to Bluesky
- **Discord Webhook**: For Discord notifications
- **Telegram API**: For Telegram posting

### 3. **Data Flow Requirements**
- **Input Validation**: Ensure RSS data structure integrity
- **URL Processing**: Handle various URL formats and tracking parameters
- **Content Generation**: AI-powered summarization with character limits
- **Error Handling**: Graceful degradation on individual node failures
- **State Management**: Persistent storage across workflow executions

### 4. **Performance Considerations**
- **Batch Size**: 3 articles per batch to avoid overwhelming APIs
- **Memory Management**: Maximum 5000 keys with automatic cleanup
- **Rate Limiting**: Controlled intervals between executions
- **Parallel Processing**: Multiple RSS feeds processed simultaneously

## Failsafe Mechanisms

### 1. **Data Structure Validation**
- **Required Fields**: Ensure text/content exists before processing
- **Fallback Values**: Default content if expected fields are missing
- **Type Checking**: Validate data types before operations

### 2. **Error Isolation**
- **Individual Node Errors**: Don't crash the entire workflow
- **Continue on Error**: `onError: continueRegularOutput` for all social media nodes
- **Graceful Degradation**: Skip problematic items but continue processing

### 3. **State Recovery**
- **Pending State Cleanup**: Remove stale reservations after 2 hours
- **TTL Enforcement**: Automatic expiration of old entries
- **Memory Limits**: Prevent unbounded growth with key count limits

### 4. **API Protection**
- **Rate Limiting**: Controlled execution intervals
- **Batch Processing**: Small batches to avoid overwhelming external APIs
- **Retry Logic**: Built-in retry mechanisms for transient failures

## Monitoring & Debugging

### 1. **Execution Logs**
- **Deduplication Statistics**: Counts of processed vs. skipped articles
- **Article Details**: Titles and deduplication keys for each item
- **Error Logging**: Detailed error information for troubleshooting

### 2. **State Inspection**
- **Posted Articles**: Current count and age of posted articles
- **Pending Articles**: Articles currently being processed
- **Memory Usage**: Total keys and cleanup statistics

### 3. **Performance Metrics**
- **Processing Time**: Time taken for each execution
- **Success Rates**: Percentage of successful posts per platform
- **Error Rates**: Frequency and types of errors encountered

## Configuration Parameters

### 1. **Deduplication Settings**
- **TTL**: 30 days (configurable)
- **Pending TTL**: 2 hours (configurable)
- **Max Keys**: 5000 (configurable)
- **Max Per Run**: 0 (unlimited, configurable)

### 2. **Processing Settings**
- **Batch Size**: 3 articles per batch
- **Wait Intervals**: 36 minutes (short), 120 minutes (long)
- **Trigger Intervals**: 4, 6, 8, 10 minutes for different feeds

### 3. **Content Settings**
- **Character Limit**: 200 characters for social media posts
- **Fallback Text**: "RSS Article Update" for missing content
- **URL Shortening**: Automatic TinyURL generation

## Expected Behavior

### 1. **Normal Operation**
- RSS feeds are polled at specified intervals
- New articles are deduplicated against 30-day history
- AI generates social media content with character limits
- Content is posted to all platforms simultaneously
- Success is confirmed and state is updated

### 2. **Error Scenarios**
- **Missing Content**: Fallback text is used
- **API Failures**: Individual platform failures don't stop others
- **Data Corruption**: Malformed items are skipped with logging
- **Rate Limiting**: Wait nodes provide controlled delays

### 3. **Recovery Scenarios**
- **Workflow Interruption**: Pending state prevents duplicate processing
- **Node Failures**: Error handling continues processing
- **Memory Issues**: Automatic cleanup prevents bloat
- **Credential Expiry**: Clear error messages for authentication issues

## Success Criteria

The recreated workflow should:
1. **Process multiple RSS feeds** without conflicts
2. **Prevent duplicate posts** for 30 days
3. **Generate AI content** within character limits
4. **Post to all platforms** with error isolation
5. **Maintain state** across executions
6. **Provide comprehensive logging** for monitoring
7. **Handle errors gracefully** without workflow crashes
8. **Scale efficiently** with automatic cleanup and memory management

This specification provides all necessary details for an AI agent to recreate the complete n8n RSS processing workflow with enterprise-grade reliability and performance.
