# Troubleshooting Guide

## Common Errors and Solutions

### 1. Spread Syntax Error in Mastodon Node

**Error Message:**
```
TypeError: Spread syntax requires ...iterable[Symbol.iterator] to be a function
```

**Root Cause:**
This error occurs when the Mastodon node (or other social media nodes) receives data that doesn't have the expected structure. The node expects a specific data format but receives something that can't be spread/iterated.

**What Was Fixed:**
1. **Added Data Transformation Node**: A new "Data Transform" node was added after the LLM Chain to ensure data structure consistency
2. **Improved Error Handling**: Added fallback values for text fields in all social media nodes
3. **Data Validation**: The transformation node validates and cleans the data before sending to social media nodes

**New Flow:**
```
LLM Chain → Data Transform → Social Media Nodes
```

**Data Transformation Logic:**
```javascript
// Ensures required structure exists
const text = item.json?.text || item.json?.content || 'No content available';

// Creates clean, validated data structure
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
```

### 2. Fallback Values Added

All social media nodes now have fallback values:
- **Mastodon**: `{{ $json.text || $json.content || 'RSS Article Update' }}`
- **Bluesky**: `{{ $json.text || $json.content || 'RSS Article Update' }}`
- **Discord**: `{{ $json.text || $json.content || 'RSS Article Update' }}`
- **Telegram**: `{{ $json.text || $json.content || 'RSS Article Update' }}`

### 3. Error Prevention Strategies

1. **Data Validation**: Check data structure before processing
2. **Fallback Values**: Provide default content if expected fields are missing
3. **Error Handling**: Use try-catch blocks to handle malformed data gracefully
4. **Logging**: Log errors for debugging without stopping the workflow

## How to Test the Fix

1. **Import the updated workflow** into n8n
2. **Run a test execution** with the manual trigger
3. **Check the execution logs** for any remaining errors
4. **Verify data flow** through the Data Transform node

## Prevention Tips

1. **Always validate data** before sending to external APIs
2. **Use fallback values** for critical fields
3. **Test with various data formats** to ensure robustness
4. **Monitor execution logs** for early error detection
5. **Keep workflow versions** in version control (like this GitHub repo)

## If Errors Persist

1. Check the **Data Transform node logs** for data structure issues
2. Verify **RSS feed data** is in expected format
3. Test **individual social media nodes** with simple data
4. Review **n8n version compatibility** (tested with 1.106.3)
5. Check **credentials and API endpoints** are correct
