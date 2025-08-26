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
2. **Added Data Validation Node**: A robust validation node that sanitizes data before sending to social media nodes
3. **Added Debug Final Node**: A debugging node that logs the exact data structure being sent to social media nodes
4. **Improved Error Handling**: Added fallback values for text fields in all social media nodes
5. **Safe Data Processing**: Eliminated unsafe spread syntax and added type checking

**New Flow:**
```
LLM Chain → Data Transform → Data Validation → Debug Final → Social Media Nodes
```

**Key Improvements:**
- **Type Safety**: All data is converted to strings before processing
- **Structure Validation**: Ensures json property exists and is an object
- **Field Filtering**: Only includes safe, string-based fields
- **Comprehensive Logging**: Shows exactly what data is being processed
- **Error Isolation**: Skips problematic items without crashing the workflow

**Data Validation Logic:**
```javascript
// Final data validation and sanitization for social media nodes
const text = String(item.json.text || item.json.content || 'RSS Article Update');
const title = String(item.json.title || 'RSS Article');
const link = String(item.json.link || '');
const dedupe_key = String(item.json.dedupe_key || '');

// Create a completely clean, validated item
const validatedItem = {
  json: {
    text: text,
    title: title,
    link: link,
    dedupe_key: dedupe_key,
    content: text,
    // Only include safe, string-based fields
    ...Object.fromEntries(
      Object.entries(item.json)
        .filter(([key, value]) => 
          value !== undefined && 
          value !== null && 
          typeof value === 'string' &&
          key !== 'text' && 
          key !== 'title' && 
          key !== 'link' && 
          key !== 'dedupe_key' && 
          key !== 'content'
        )
    )
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
4. **Verify data flow** through all validation nodes:
   - Data Transform → Data Validation → Debug Final → Social Media Nodes
5. **Look for debug logs** showing the exact data structure being processed

## Prevention Tips

1. **Always validate data** before sending to external APIs
2. **Use fallback values** for critical fields
3. **Test with various data formats** to ensure robustness
4. **Monitor execution logs** for early error detection
5. **Keep workflow versions** in version control (like this GitHub repo)

## If Errors Persist

1. Check the **Debug Final node logs** for the exact data structure being sent to social media nodes
2. Verify **Data Validation node logs** for any items being skipped
3. Check **Data Transform node logs** for initial transformation issues
4. Verify **RSS feed data** is in expected format
5. Test **individual social media nodes** with simple data
6. Review **n8n version compatibility** (tested with 1.106.3)
7. Check **credentials and API endpoints** are correct
