# Common Patterns - JavaScript Code Node

Production-tested patterns for n8n Code nodes. These patterns are proven in real workflows.

---

## Overview

This guide covers the 10 most useful Code node patterns for n8n workflows. Each pattern includes:
- **Use Case**: When to use this pattern
- **Key Techniques**: Important coding techniques demonstrated
- **Complete Example**: Working code you can adapt
- **Variations**: Common modifications

**Pattern Categories:**
- Data Aggregation (Patterns 1, 5, 10)
- Content Processing (Patterns 2, 3)
- Data Validation & Comparison (Patterns 4)
- Data Transformation (Patterns 5, 6, 7)
- Output Formatting (Pattern 8)
- Filtering & Ranking (Pattern 9)

---

## Pattern 1: Multi-Source Data Aggregation

**Use Case**: Combining data from multiple APIs, RSS feeds, webhooks, or databases

**When to use:**
- Collecting data from multiple services
- Normalizing different API response formats
- Merging data sources into unified structure
- Building aggregated reports

**Key Techniques**: Loop iteration, conditional parsing, data normalization

### Complete Example

```javascript
// Process and structure data collected from multiple sources
const allItems = $input.all();
let processedArticles = [];

// Handle different source formats
for (const item of allItems) {
  const sourceName = item.json.name || 'Unknown';
  const sourceData = item.json;

  // Parse source-specific structure - Hacker News format
  if (sourceName === 'Hacker News' && sourceData.hits) {
    for (const hit of sourceData.hits) {
      processedArticles.push({
        title: hit.title,
        url: hit.url,
        summary: hit.story_text || 'No summary',
        source: 'Hacker News',
        score: hit.points || 0,
        fetchedAt: new Date().toISOString()
      });
    }
  }

  // Parse source-specific structure - Reddit format
  else if (sourceName === 'Reddit' && sourceData.data?.children) {
    for (const post of sourceData.data.children) {
      processedArticles.push({
        title: post.data.title,
        url: post.data.url,
        summary: post.data.selftext || 'No summary',
        source: 'Reddit',
        score: post.data.score || 0,
        fetchedAt: new Date().toISOString()
      });
    }
  }

  // Parse source-specific structure - RSS feed format
  else if (sourceName === 'RSS' && sourceData.items) {
    for (const rssItem of sourceData.items) {
      processedArticles.push({
        title: rssItem.title,
        url: rssItem.link,
        summary: rssItem.description || 'No summary',
        source: 'RSS Feed',
        score: 0,
        fetchedAt: new Date().toISOString()
      });
    }
  }
}

// Sort by score (highest first)
processedArticles.sort((a, b) => b.score - a.score);

return processedArticles.map(article => ({json: article}));
```

### Variations

```javascript
// Variation 1: Add source weighting
for (const article of processedArticles) {
  const weights = {
    'Hacker News': 1.5,
    'Reddit': 1.0,
    'RSS Feed': 0.8
  };

  article.weightedScore = article.score * (weights[article.source] || 1.0);
}

// Variation 2: Filter by minimum score
processedArticles = processedArticles.filter(article => article.score >= 10);

// Variation 3: Deduplicate by URL
const seen = new Set();
processedArticles = processedArticles.filter(article => {
  if (seen.has(article.url)) {
    return false;
  }
  seen.add(article.url);
  return true;
});
```

---

## Pattern 2: Regex Filtering & Pattern Matching

**Use Case**: Content analysis, keyword extraction, mention tracking, text parsing

**When to use:**
- Extracting mentions or tags from text
- Finding patterns in unstructured data
- Counting keyword occurrences
- Validating formats (emails, phone numbers)

**Key Techniques**: Regex matching, object aggregation, sorting/ranking

### Complete Example

```javascript
// Extract and track mentions using regex patterns
const etfPattern = /\b([A-Z]{2,5})\b/g;
const knownETFs = ['VOO', 'VTI', 'VT', 'SCHD', 'QYLD', 'VXUS', 'SPY', 'QQQ'];

const etfMentions = {};

for (const item of $input.all()) {
  const data = item.json.data;

  // Skip if no data or children
  if (!data?.children) continue;

  for (const post of data.children) {
    // Combine title and body text
    const title = post.data.title || '';
    const body = post.data.selftext || '';
    const combinedText = (title + ' ' + body).toUpperCase();

    // Find all matches
    const matches = combinedText.match(etfPattern);

    if (matches) {
      for (const match of matches) {
        // Only count known ETFs
        if (knownETFs.includes(match)) {
          if (!etfMentions[match]) {
            etfMentions[match] = {
              count: 0,
              totalScore: 0,
              posts: []
            };
          }

          etfMentions[match].count++;
          etfMentions[match].totalScore += post.data.score || 0;
          etfMentions[match].posts.push({
            title: post.data.title,
            url: post.data.url,
            score: post.data.score
          });
        }
      }
    }
  }
}

// Convert to array and sort by mention count
return Object.entries(etfMentions)
  .map(([etf, data]) => ({
    json: {
      etf,
      mentions: data.count,
      totalScore: data.totalScore,
      averageScore: data.totalScore / data.count,
      topPosts: data.posts
        .sort((a, b) => b.score - a.score)
        .slice(0, 3)
    }
  }))
  .sort((a, b) => b.json.mentions - a.json.mentions);
```

### Variations

```javascript
// Variation 1: Email extraction
const emailPattern = /\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b/g;
const emails = text.match(emailPattern) || [];

// Variation 2: Phone number extraction
const phonePattern = /\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/g;
const phones = text.match(phonePattern) || [];

// Variation 3: Hashtag extraction
const hashtagPattern = /#(\w+)/g;
const hashtags = [];
let match;
while ((match = hashtagPattern.exec(text)) !== null) {
  hashtags.push(match[1]);
}

// Variation 4: URL extraction
const urlPattern = /https?:\/\/[^\s]+/g;
const urls = text.match(urlPattern) || [];
```

---

## Pattern 3: Markdown Parsing & Structured Data Extraction

**Use Case**: Parsing formatted text, extracting structured fields, content transformation

**When to use:**
- Parsing markdown or HTML
- Extracting data from structured text
- Converting formatted content to JSON
- Processing documentation or articles

**Key Techniques**: Regex grouping, helper functions, data normalization, while loops for iteration

### Complete Example

```javascript
// Parse markdown and extract structured information
const markdown = $input.first().json.data.markdown;
const adRegex = /##\s*(.*?)\n(.*?)(?=\n##|\n---|$)/gs;

const ads = [];
let match;

// Helper function to parse time strings to minutes
function parseTimeToMinutes(timeStr) {
  if (!timeStr) return 999999;  // Sort unparseable times last

  const hourMatch = timeStr.match(/(\d+)\s*hour/);
  const dayMatch = timeStr.match(/(\d+)\s*day/);
  const minMatch = timeStr.match(/(\d+)\s*min/);

  let totalMinutes = 0;
  if (dayMatch) totalMinutes += parseInt(dayMatch[1]) * 1440;  // 24 * 60
  if (hourMatch) totalMinutes += parseInt(hourMatch[1]) * 60;
  if (minMatch) totalMinutes += parseInt(minMatch[1]);

  return totalMinutes;
}

// Extract all job postings from markdown
while ((match = adRegex.exec(markdown)) !== null) {
  const title = match[1]?.trim() || 'No title';
  const content = match[2]?.trim() || '';

  // Extract structured fields from content
  const districtMatch = content.match(/\*\*District:\*\*\s*(.*?)(?:\n|$)/);
  const salaryMatch = content.match(/\*\*Salary:\*\*\s*(.*?)(?:\n|$)/);
  const timeMatch = content.match(/Posted:\s*(.*?)\*/);

  ads.push({
    title: title,
    district: districtMatch?.[1].trim() || 'Unknown',
    salary: salaryMatch?.[1].trim() || 'Not specified',
    postedTimeAgo: timeMatch?.[1] || 'Unknown',
    timeInMinutes: parseTimeToMinutes(timeMatch?.[1]),
    fullContent: content,
    extractedAt: new Date().toISOString()
  });
}

// Sort by recency (posted time)
ads.sort((a, b) => a.timeInMinutes - b.timeInMinutes);

return ads.map(ad => ({json: ad}));
```

### Variations

```javascript
// Variation 1: Parse HTML table to JSON
const tableRegex = /<tr>(.*?)<\/tr>/gs;
const cellRegex = /<td>(.*?)<\/td>/g;

const rows = [];
let tableMatch;

while ((tableMatch = tableRegex.exec(htmlTable)) !== null) {
  const cells = [];
  let cellMatch;

  while ((cellMatch = cellRegex.exec(tableMatch[1])) !== null) {
    cells.push(cellMatch[1].trim());
  }

  if (cells.length > 0) {
    rows.push(cells);
  }
}

// Variation 2: Extract code blocks from markdown
const codeBlockRegex = /```(\w+)?\n(.*?)```/gs;
const codeBlocks = [];

while ((match = codeBlockRegex.exec(markdown)) !== null) {
  codeBlocks.push({
    language: match[1] || 'plain',
    code: match[2].trim()
  });
}

// Variation 3: Parse YAML frontmatter
const frontmatterRegex = /^---\n(.*?)\n---/s;
const frontmatterMatch = content.match(frontmatterRegex);

if (frontmatterMatch) {
  const yamlLines = frontmatterMatch[1].split('\n');
  const metadata = {};

  for (const line of yamlLines) {
    const [key, ...valueParts] = line.split(':');
    if (key && valueParts.length > 0) {
      metadata[key.trim()] = valueParts.join(':').trim();
    }
  }
}
```

---

## Pattern 4: JSON Comparison & Validation

**Use Case**: Workflow versioning, configuration validation, change detection, data integrity

**When to use:**
- Comparing two versions of data
- Detecting changes in configurations
- Validating data consistency
- Checking for differences

**Key Techniques**: JSON ordering, base64 decoding, deep comparison, object manipulation

### Complete Example

```javascript
// Compare and validate JSON objects from different sources
const orderJsonKeys = (jsonObj) => {
  const ordered = {};
  Object.keys(jsonObj).sort().forEach(key => {
    ordered[key] = jsonObj[key];
  });
  return ordered;
};

const allItems = $input.all();

// Assume first item is base64-encoded original, second is current
const origWorkflow = JSON.parse(
  Buffer.from(allItems[0].json.content, 'base64').toString()
);
const currentWorkflow = allItems[1].json;

// Order keys for consistent comparison
const orderedOriginal = orderJsonKeys(origWorkflow);
const orderedCurrent = orderJsonKeys(currentWorkflow);

// Deep comparison
const isSame = JSON.stringify(orderedOriginal) === JSON.stringify(orderedCurrent);

// Find differences
const differences = [];
for (const key of Object.keys(orderedOriginal)) {
  if (JSON.stringify(orderedOriginal[key]) !== JSON.stringify(orderedCurrent[key])) {
    differences.push({
      field: key,
      original: orderedOriginal[key],
      current: orderedCurrent[key]
    });
  }
}

// Check for new keys
for (const key of Object.keys(orderedCurrent)) {
  if (!(key in orderedOriginal)) {
    differences.push({
      field: key,
      original: null,
      current: orderedCurrent[key],
      status: 'new'
    });
  }
}

return [{
  json: {
    identical: isSame,
    differenceCount: differences.length,
    differences: differences,
    original: orderedOriginal,
    current: orderedCurrent,
    comparedAt: new Date().toISOString()
  }
}];
```

### Variations

```javascript
// Variation 1: Simple equality check
const isEqual = JSON.stringify(obj1) === JSON.stringify(obj2);

// Variation 2: Deep diff with detailed changes
function deepDiff(obj1, obj2, path = '') {
  const changes = [];

  for (const key in obj1) {
    const currentPath = path ? `${path}.${key}` : key;

    if (!(key in obj2)) {
      changes.push({type: 'removed', path: currentPath, value: obj1[key]});
    } else if (typeof obj1[key] === 'object' && typeof obj2[key] === 'object') {
      changes.push(...deepDiff(obj1[key], obj2[key], currentPath));
    } else if (obj1[key] !== obj2[key]) {
      changes.push({
        type: 'modified',
        path: currentPath,
        from: obj1[key],
        to: obj2[key]
      });
    }
  }

  for (const key in obj2) {
    if (!(key in obj1)) {
      const currentPath = path ? `${path}.${key}` : key;
      changes.push({type: 'added', path: currentPath, value: obj2[key]});
    }
  }

  return changes;
}

// Variation 3: Schema validation
function validateSchema(data, schema) {
  const errors = [];

  for (const field of schema.required || []) {
    if (!(field in data)) {
      errors.push(`Missing required field: ${field}`);
    }
  }

  for (const [field, type] of Object.entries(schema.types || {})) {
    if (field in data && typeof data[field] !== type) {
      errors.push(`Field ${field} should be ${type}, got ${typeof data[field]}`);
    }
  }

  return {
    valid: errors.length === 0,
    errors
  };
}
```

---

## Pattern 5: CRM Data Transformation

**Use Case**: Lead enrichment, data normalization, API preparation, form data processing

**When to use:**
- Processing form submissions
- Preparing data for CRM APIs
- Normalizing contact information
- Enriching lead data

**Key Techniques**: Object destructuring, data mapping, format conversion, field splitting

### Complete Example

```javascript
// Transform form data into CRM-compatible format
const item = $input.all()[0];
const {
  name,
  email,
  phone,
  company,
  course_interest,
  message,
  timestamp
} = item.json;

// Split name into first and last
const nameParts = name.split(' ');
const firstName = nameParts[0] || '';
const lastName = nameParts.slice(1).join(' ') || 'Unknown';

// Format phone number
const cleanPhone = phone.replace(/[^\d]/g, '');  // Remove non-digits

// Build CRM data structure
const crmData = {
  data: {
    type: 'Contact',
    attributes: {
      first_name: firstName,
      last_name: lastName,
      email1: email,
      phone_work: cleanPhone,
      account_name: company,
      description: `Course Interest: ${course_interest}\n\nMessage: ${message}\n\nSubmitted: ${timestamp}`,
      lead_source: 'Website Form',
      status: 'New'
    }
  },
  metadata: {
    original_submission: timestamp,
    processed_at: new Date().toISOString()
  }
};

return [{
  json: {
    ...item.json,
    crmData,
    processed: true
  }
}];
```

### Variations

```javascript
// Variation 1: Multiple contact processing
const contacts = $input.all();

return contacts.map(item => {
  const data = item.json;
  const [firstName, ...lastNameParts] = data.name.split(' ');

  return {
    json: {
      firstName,
      lastName: lastNameParts.join(' ') || 'Unknown',
      email: data.email.toLowerCase(),
      phone: data.phone.replace(/[^\d]/g, ''),
      tags: [data.source, data.interest_level].filter(Boolean)
    }
  };
});

// Variation 2: Field validation and normalization
function normalizePContact(raw) {
  return {
    first_name: raw.firstName?.trim() || '',
    last_name: raw.lastName?.trim() || 'Unknown',
    email: raw.email?.toLowerCase().trim() || '',
    phone: raw.phone?.replace(/[^\d]/g, '') || '',
    company: raw.company?.trim() || 'Unknown',
    title: raw.title?.trim() || '',
    valid: Boolean(raw.email && raw.firstName)
  };
}

// Variation 3: Lead scoring
function calculateLeadScore(data) {
  let score = 0;

  if (data.email) score += 10;
  if (data.phone) score += 10;
  if (data.company) score += 15;
  if (data.title?.toLowerCase().includes('director')) score += 20;
  if (data.title?.toLowerCase().includes('manager')) score += 15;
  if (data.message?.length > 100) score += 10;

  return score;
}
```

---

## Pattern 6: Release Information Processing

**Use Case**: Version management, changelog parsing, release notes generation, GitHub API processing

**When to use:**
- Processing GitHub releases
- Filtering stable versions
- Generating changelog summaries
- Extracting version information

**Key Techniques**: Array filtering, conditional field extraction, date formatting, string manipulation

### Complete Example

```javascript
// Extract and filter stable releases from GitHub API
const allReleases = $input.first().json;

const stableReleases = allReleases
  .filter(release => !release.prerelease && !release.draft)
  .slice(0, 10)
  .map(release => {
    // Extract highlights section from changelog
    const body = release.body || '';
    let highlights = 'No highlights available';

    if (body.includes('## Highlights:')) {
      highlights = body.split('## Highlights:')[1]?.split('##')[0]?.trim();
    } else {
      // Fallback to first 500 chars
      highlights = body.substring(0, 500) + '...';
    }

    return {
      tag: release.tag_name,
      name: release.name,
      published: release.published_at,
      publishedDate: new Date(release.published_at).toLocaleDateString(),
      author: release.author.login,
      url: release.html_url,
      changelog: body,
      highlights: highlights,
      assetCount: release.assets.length,
      assets: release.assets.map(asset => ({
        name: asset.name,
        size: asset.size,
        downloadCount: asset.download_count,
        downloadUrl: asset.browser_download_url
      }))
    };
  });

return stableReleases.map(release => ({json: release}));
```

### Variations

```javascript
// Variation 1: Version comparison
function compareVersions(v1, v2) {
  const parts1 = v1.replace('v', '').split('.').map(Number);
  const parts2 = v2.replace('v', '').split('.').map(Number);

  for (let i = 0; i < Math.max(parts1.length, parts2.length); i++) {
    const num1 = parts1[i] || 0;
    const num2 = parts2[i] || 0;

    if (num1 > num2) return 1;
    if (num1 < num2) return -1;
  }

  return 0;
}

// Variation 2: Breaking change detection
function hasBreakingChanges(changelog) {
  const breakingKeywords = [
    'BREAKING CHANGE',
    'breaking change',
    'BC:',
    '💥'
  ];

  return breakingKeywords.some(keyword => changelog.includes(keyword));
}

// Variation 3: Extract version numbers
const versionPattern = /v?(\d+)\.(\d+)\.(\d+)/;
const match = tagName.match(versionPattern);

if (match) {
  const [_, major, minor, patch] = match;
  const version = {major: parseInt(major), minor: parseInt(minor), patch: parseInt(patch)};
}
```

---

## Pattern 7: Array Transformation with Context

**Use Case**: Quick data transformation, field mapping, adding computed fields

**When to use:**
- Transforming arrays with additional context
- Adding calculated fields
- Simplifying complex objects
- Pluralization logic

**Key Techniques**: Array methods chaining, ternary operators, computed properties

### Complete Example

```javascript
// Transform releases with contextual information
const releases = $input.first().json
  .filter(release => !release.prerelease && !release.draft)
  .slice(0, 10)
  .map(release => ({
    version: release.tag_name,
    assetCount: release.assets.length,
    assetsCountText: release.assets.length === 1 ? 'file' : 'files',
    downloadUrl: release.html_url,
    isRecent: new Date(release.published_at) > new Date(Date.now() - 30 * 24 * 60 * 60 * 1000),
    age: Math.floor((Date.now() - new Date(release.published_at)) / (24 * 60 * 60 * 1000)),
    ageText: `${Math.floor((Date.now() - new Date(release.published_at)) / (24 * 60 * 60 * 1000))} days ago`
  }));

return releases.map(release => ({json: release}));
```

### Variations

```javascript
// Variation 1: Add ranking
const items = $input.all()
  .sort((a, b) => b.json.score - a.json.score)
  .map((item, index) => ({
    json: {
      ...item.json,
      rank: index + 1,
      medal: index < 3 ? ['🥇', '🥈', '🥉'][index] : ''
    }
  }));

// Variation 2: Add percentage calculations
const total = $input.all().reduce((sum, item) => sum + item.json.value, 0);

const itemsWithPercentage = $input.all().map(item => ({
  json: {
    ...item.json,
    percentage: ((item.json.value / total) * 100).toFixed(2) + '%'
  }
}));

// Variation 3: Add category labels
const categorize = (value) => {
  if (value > 100) return 'High';
  if (value > 50) return 'Medium';
  return 'Low';
};

const categorized = $input.all().map(item => ({
  json: {
    ...item.json,
    category: categorize(item.json.value)
  }
}));
```

---

## Pattern 8: Slack Block Kit Formatting

**Use Case**: Chat notifications, rich message formatting, interactive messages

**When to use:**
- Sending formatted Slack messages
- Creating interactive notifications
- Building rich content for chat platforms
- Status reports and alerts

**Key Techniques**: Template literals, nested objects, Block Kit syntax, date formatting

### Complete Example

```javascript
// Create Slack-formatted message with structured blocks
const date = new Date().toISOString().split('T')[0];
const data = $input.first().json;

return [{
  json: {
    text: `Daily Report - ${date}`,  // Fallback text
    blocks: [
      {
        type: "header",
        text: {
          type: "plain_text",
          text: `📊 Daily Security Report - ${date}`
        }
      },
      {
        type: "section",
        text: {
          type: "mrkdwn",
          text: `*Status:* ${data.status === 'ok' ? '✅ All Clear' : '⚠️ Issues Detected'}\n*Alerts:* ${data.alertCount || 0}\n*Updated:* ${new Date().toLocaleString()}`
        }
      },
      {
        type: "divider"
      },
      {
        type: "section",
        fields: [
          {
            type: "mrkdwn",
            text: `*Failed Logins:*\n${data.failedLogins || 0}`
          },
          {
            type: "mrkdwn",
            text: `*API Errors:*\n${data.apiErrors || 0}`
          },
          {
            type: "mrkdwn",
            text: `*Uptime:*\n${data.uptime || '100%'}`
          },
          {
            type: "mrkdwn",
            text: `*Response Time:*\n${data.avgResponseTime || 'N/A'}ms`
          }
        ]
      },
      {
        type: "context",
        elements: [{
          type: "mrkdwn",
          text: `Report generated automatically by n8n workflow`
        }]
      }
    ]
  }
}];
```

### Variations

```javascript
// Variation 1: Interactive buttons
const blocksWithButtons = [
  {
    type: "section",
    text: {
      type: "mrkdwn",
      text: "Would you like to approve this request?"
    },
    accessory: {
      type: "button",
      text: {
        type: "plain_text",
        text: "Approve"
      },
      style: "primary",
      value: "approve",
      action_id: "approve_button"
    }
  }
];

// Variation 2: List formatting
const items = ['Item 1', 'Item 2', 'Item 3'];
const formattedList = items.map((item, i) => `${i + 1}. ${item}`).join('\n');

// Variation 3: Status indicators
function getStatusEmoji(status) {
  const statusMap = {
    'success': '✅',
    'warning': '⚠️',
    'error': '❌',
    'info': 'ℹ️'
  };

  return statusMap[status] || '•';
}

// Variation 4: Truncate long messages
function truncate(text, maxLength = 3000) {
  if (text.length <= maxLength) return text;
  return text.substring(0, maxLength - 3) + '...';
}
```

---

## Pattern 9: Top N Filtering & Ranking

**Use Case**: RAG pipelines, ranking algorithms, result filtering, leaderboards

**When to use:**
- Getting top results by score
- Filtering best/worst performers
- Building leaderboards
- Relevance ranking

**Key Techniques**: Sorting, slicing, null coalescing, score calculations

### Complete Example

```javascript
// Filter and rank by similarity score, return top results
const ragResponse = $input.item.json;
const chunks = ragResponse.chunks || [];

// Sort by similarity (highest first)
const topChunks = chunks
  .sort((a, b) => (b.similarity || 0) - (a.similarity || 0))
  .slice(0, 6);

return [{
  json: {
    query: ragResponse.query,
    topChunks: topChunks,
    count: topChunks.length,
    maxSimilarity: topChunks[0]?.similarity || 0,
    minSimilarity: topChunks[topChunks.length - 1]?.similarity || 0,
    averageSimilarity: topChunks.reduce((sum, chunk) => sum + (chunk.similarity || 0), 0) / topChunks.length
  }
}];
```

### Variations

```javascript
// Variation 1: Top N with minimum threshold
const threshold = 0.7;
const topItems = $input.all()
  .filter(item => item.json.score >= threshold)
  .sort((a, b) => b.json.score - a.json.score)
  .slice(0, 10);

// Variation 2: Bottom N (worst performers)
const bottomItems = $input.all()
  .sort((a, b) => a.json.score - b.json.score)  // Ascending
  .slice(0, 5);

// Variation 3: Top N by multiple criteria
const ranked = $input.all()
  .map(item => ({
    ...item,
    compositeScore: (item.json.relevance * 0.6) + (item.json.recency * 0.4)
  }))
  .sort((a, b) => b.compositeScore - a.compositeScore)
  .slice(0, 10);

// Variation 4: Percentile filtering
const allScores = $input.all().map(item => item.json.score).sort((a, b) => b - a);
const percentile95 = allScores[Math.floor(allScores.length * 0.05)];

const topPercentile = $input.all().filter(item => item.json.score >= percentile95);
```

---

## Pattern 10: String Aggregation & Reporting

**Use Case**: Report generation, log aggregation, content concatenation, summary creation

**When to use:**
- Combining multiple text outputs
- Generating reports from data
- Aggregating logs or messages
- Creating formatted summaries

**Key Techniques**: Array joining, string concatenation, template literals, timestamp handling

### Complete Example

```javascript
// Aggregate multiple text inputs into formatted report
const allItems = $input.all();

// Collect all messages
const messages = allItems.map(item => item.json.message);

// Build report
const header = `🎯 **Daily Summary Report**\n📅 ${new Date().toLocaleString()}\n📊 Total Items: ${messages.length}\n\n`;
const divider = '\n\n---\n\n';
const footer = `\n\n---\n\n✅ Report generated at ${new Date().toISOString()}`;

const finalReport = header + messages.join(divider) + footer;

return [{
  json: {
    report: finalReport,
    messageCount: messages.length,
    generatedAt: new Date().toISOString(),
    reportLength: finalReport.length
  }
}];
```

### Variations

```javascript
// Variation 1: Numbered list
const numberedReport = allItems
  .map((item, index) => `${index + 1}. ${item.json.title}\n   ${item.json.description}`)
  .join('\n\n');

// Variation 2: Markdown table
const headers = '| Name | Status | Score |\n|------|--------|-------|\n';
const rows = allItems
  .map(item => `| ${item.json.name} | ${item.json.status} | ${item.json.score} |`)
  .join('\n');

const table = headers + rows;

// Variation 3: HTML report
const htmlReport = `
<!DOCTYPE html>
<html>
<head><title>Report</title></head>
<body>
  <h1>Report - ${new Date().toLocaleDateString()}</h1>
  <ul>
    ${allItems.map(item => `<li>${item.json.title}: ${item.json.value}</li>`).join('\n    ')}
  </ul>
</body>
</html>
`;

// Variation 4: JSON summary
const summary = {
  generated: new Date().toISOString(),
  totalItems: allItems.length,
  items: allItems.map(item => item.json),
  statistics: {
    total: allItems.reduce((sum, item) => sum + (item.json.value || 0), 0),
    average: allItems.reduce((sum, item) => sum + (item.json.value || 0), 0) / allItems.length,
    max: Math.max(...allItems.map(item => item.json.value || 0)),
    min: Math.min(...allItems.map(item => item.json.value || 0))
  }
};
```

---

## Choosing the Right Pattern

### Pattern Selection Guide

| Your Goal | Use Pattern |
|-----------|-------------|
| Combine multiple API responses | Pattern 1 (Multi-source Aggregation) |
| Extract mentions or keywords | Pattern 2 (Regex Filtering) |
| Parse formatted text | Pattern 3 (Markdown Parsing) |
| Detect changes in data | Pattern 4 (JSON Comparison) |
| Prepare form data for CRM | Pattern 5 (CRM Transformation) |
| Process GitHub releases | Pattern 6 (Release Processing) |
| Add computed fields | Pattern 7 (Array Transformation) |
| Format Slack messages | Pattern 8 (Block Kit Formatting) |
| Get top results | Pattern 9 (Top N Filtering) |
| Create text reports | Pattern 10 (String Aggregation) |

### Combining Patterns

Many real workflows combine multiple patterns:

```javascript
// Example: Multi-source aggregation + Top N filtering
const allItems = $input.all();
const aggregated = [];

// Pattern 1: Aggregate from different sources
for (const item of allItems) {
  // ... aggregation logic
  aggregated.push(normalizedItem);
}

// Pattern 9: Get top 10 by score
const top10 = aggregated
  .sort((a, b) => b.score - a.score)
  .slice(0, 10);

// Pattern 10: Generate report
const report = `Top 10 Items:\n\n${top10.map((item, i) => `${i + 1}. ${item.title} (${item.score})`).join('\n')}`;

return [{json: {report, items: top10}}];
```

---

## Summary

**Most Useful Patterns**:
1. Multi-source Aggregation - Combining data from APIs, databases
2. Top N Filtering - Rankings, leaderboards, best results
3. Data Transformation - CRM data, field mapping, enrichment

**Key Techniques Across Patterns**:
- Array methods (map, filter, reduce, sort, slice)
- Regex for pattern matching
- Object manipulation and destructuring
- Error handling with optional chaining
- Template literals for formatting

**See Also**:
- [DATA_ACCESS.md](DATA_ACCESS.md) - Data access methods
- [ERROR_PATTERNS.md](ERROR_PATTERNS.md) - Avoid common mistakes
- [BUILTIN_FUNCTIONS.md](BUILTIN_FUNCTIONS.md) - Built-in helpers
- [../n8n-production-readiness/SKILL.md](../n8n-production-readiness/) - Production hardening tiers

---

## Pattern 11: Entry-Point Validation (Tier 2+)

**Use Case**: Webhook validation, data integrity, production hardening

**When to use:**
- Receiving webhook data
- Processing client-submitted forms
- Any Tier 2+ production workflow
- Preventing silent failures

**Key Techniques**: Schema validation, null vs empty string handling, error responses

### Complete Example

```javascript
// Full entry-point validation for Tier 2 workflows
// Place immediately after Webhook node
const body = $json.body || {};
const headers = $json.headers || {};

const validation = {
  isValid: true,
  errors: [],
  warnings: []
};

// === REQUIRED FIELDS ===
const requiredFields = ['email', 'orderId', 'items'];
for (const field of requiredFields) {
  if (body[field] === undefined || body[field] === null) {
    validation.isValid = false;
    validation.errors.push(`Missing required field: ${field}`);
  }
}

// === NULL vs EMPTY STRING ===
// This is the #1 production bug!
if (body.email === null) {
  validation.isValid = false;
  validation.errors.push('email cannot be null (expected string or empty)');
}

// === TYPE CHECKS ===
if (body.email && typeof body.email !== 'string') {
  validation.isValid = false;
  validation.errors.push('email must be a string');
}

if (body.items && !Array.isArray(body.items)) {
  validation.isValid = false;
  validation.errors.push('items must be an array');
}

if (body.quantity && typeof body.quantity !== 'number') {
  validation.isValid = false;
  validation.errors.push('quantity must be a number');
}

// === FORMAT VALIDATION ===
if (body.email && typeof body.email === 'string' && !body.email.includes('@')) {
  validation.isValid = false;
  validation.errors.push('email format is invalid');
}

// === AUTHENTICATION CHECK ===
if (!headers['x-api-key']) {
  validation.isValid = false;
  validation.errors.push('Missing x-api-key header');
}

return [{
  json: {
    ...body,
    _validation: validation,
    _validatedAt: new Date().toISOString()
  }
}];
```

**After validation, use IF node:**
- True branch: `{{$json._validation.isValid}}` equals `true` → Continue processing
- False branch: → Error Response node (see Pattern 13)

### Variations

```javascript
// Variation 1: Validate nested objects
if (body.address && typeof body.address !== 'object') {
  validation.errors.push('address must be an object');
} else if (body.address) {
  if (!body.address.street) validation.errors.push('address.street is required');
  if (!body.address.city) validation.errors.push('address.city is required');
}

// Variation 2: Validate arrays of objects
if (Array.isArray(body.items)) {
  body.items.forEach((item, index) => {
    if (!item.sku) validation.errors.push(`items[${index}].sku is required`);
    if (typeof item.quantity !== 'number') {
      validation.errors.push(`items[${index}].quantity must be a number`);
    }
  });
}

// Variation 3: With warnings (non-blocking)
if (!body.phone) {
  validation.warnings.push('phone not provided - SMS notifications disabled');
}
```

---

## Pattern 12: External Logging (Tier 2+)

**Use Case**: Debugging production issues, audit trails, workflow observability

**When to use:**
- Tier 2+ workflows
- Debugging "silent failures"
- Building audit trails
- Any time you need to see what data is coming in

**Key Techniques**: Structured logging, sensitive data exclusion, stage tracking

### Complete Example

```javascript
// Structured log entry - use at key workflow stages
// Stage options: 'entry', 'validation', 'processing', 'response', 'error'
const logEntry = {
  workflow_id: $workflow.id,
  workflow_name: $workflow.name,
  execution_id: $execution.id,
  log_type: 'input',    // 'input', 'decision', 'output', 'error'
  stage: 'entry',
  payload: {
    body: $json.body,
    headers: {
      'content-type': $json.headers?.['content-type'],
      'user-agent': $json.headers?.['user-agent'],
      // NEVER log: authorization, x-api-key, cookies
    }
  },
  metadata: {
    timestamp: new Date().toISOString(),
    nodeId: $node.id
  }
};

return [{json: {...$json, _logEntry: logEntry}}];
```

**Then connect to Supabase/Postgres node** to persist the log.

### Log at Each Stage

```javascript
// Entry log (after webhook)
const entryLog = {
  log_type: 'input',
  stage: 'entry',
  payload: { body: $json.body, headers: sanitizedHeaders }
};

// Validation log (after validation node)
const validationLog = {
  log_type: 'decision',
  stage: 'validation',
  payload: {
    isValid: $json._validation.isValid,
    errors: $json._validation.errors
  }
};

// Processing log (at key decision points)
const processingLog = {
  log_type: 'decision',
  stage: 'processing',
  payload: {
    action: 'order_created',
    orderId: $json.orderId
  }
};

// Response log (before sending response)
const responseLog = {
  log_type: 'output',
  stage: 'response',
  payload: {
    statusCode: 200,
    responseBody: { /* sanitized response */ }
  }
};
```

### Error Logging (Error Trigger workflow)

```javascript
// In Error Trigger workflow
const errorLog = {
  workflow_id: $json.workflow?.id,
  execution_id: $json.execution?.id,
  log_type: 'error',
  stage: 'error_handler',
  payload: {
    error_message: $json.execution?.error?.message,
    error_node: $json.execution?.error?.node?.name,
    error_stack: $json.execution?.error?.stack
  },
  metadata: {
    timestamp: new Date().toISOString()
  }
};

return [{json: errorLog}];
```

---

## Pattern 13: HTTP Error Responses (Tier 2+)

**Use Case**: Proper API responses, client feedback, debugging support

**When to use:**
- Building webhook APIs
- Any workflow that needs to communicate failure reasons
- Tier 2+ production workflows

**Key Techniques**: Status codes, structured error bodies, reference IDs

### Complete Example

```javascript
// Validation error response (400)
function validationErrorResponse(errors) {
  return [{
    json: {
      statusCode: 400,
      body: {
        status: 'error',
        code: 'VALIDATION_ERROR',
        message: 'Invalid request data',
        errors: errors  // Array of specific issues
      }
    }
  }];
}

// Authentication error response (401)
function authErrorResponse() {
  return [{
    json: {
      statusCode: 401,
      body: {
        status: 'error',
        code: 'UNAUTHORIZED',
        message: 'Invalid or missing API key'
      }
    }
  }];
}

// Not found response (404)
function notFoundResponse(resourceType, resourceId) {
  return [{
    json: {
      statusCode: 404,
      body: {
        status: 'error',
        code: 'NOT_FOUND',
        message: `${resourceType} not found`,
        resourceId: resourceId
      }
    }
  }];
}

// Internal error response (500)
function internalErrorResponse(executionId) {
  return [{
    json: {
      statusCode: 500,
      body: {
        status: 'error',
        code: 'INTERNAL_ERROR',
        message: 'An unexpected error occurred',
        referenceId: executionId  // For support tickets
      }
    }
  }];
}

// Usage based on validation result
const validation = $json._validation;

if (!validation.isValid) {
  return validationErrorResponse(validation.errors);
}

// Continue with processing...
```

### Webhook Response Node Configuration

Configure the Respond to Webhook node:
- **Respond With**: JSON
- **Status Code**: `={{$json.statusCode}}`
- **Response Body**: `={{JSON.stringify($json.body)}}`

---

## Pattern 14: Null vs Empty String Handling

**Use Case**: Preventing the #1 production bug in n8n workflows

**When to use:**
- Processing external data (webhooks, APIs)
- Any field that might come from user input
- Before comparing or storing values

**Key Techniques**: Explicit null checks, nullish coalescing, type guards

### The Problem

```javascript
// A field can be:
// - undefined  → key doesn't exist
// - null       → key exists, value is null
// - ''         → key exists, value is empty string
// - 'value'    → key exists, has value

// ❌ DANGER: These all behave differently!
const value = body.email;           // Could be any of the above
const check1 = !value;              // true for undefined, null, ''
const check2 = value == null;       // true for undefined, null (not '')
const check3 = value === null;      // true ONLY for null
const check4 = value === '';        // true ONLY for empty string
```

### Complete Example

```javascript
// Explicit null handling patterns

// Pattern A: Reject null, accept empty string
if (body.email === null) {
  return [{json: {error: 'email cannot be null', statusCode: 400}}];
}
const email = body.email || '';  // Now safe: undefined → '', keeps ''

// Pattern B: Normalize null to empty string
const email = body.email ?? '';  // null/undefined → '', keeps ''

// Pattern C: Check for "real" value (not null, not undefined, not '')
const hasEmail = body.email != null && body.email !== '';

// Pattern D: Full type guard
function isValidString(value) {
  return typeof value === 'string' && value.length > 0;
}

if (!isValidString(body.email)) {
  return [{json: {error: 'email must be a non-empty string'}}];
}
```

### Comparison Table

| Value | `!value` | `value == null` | `value === null` | `value === ''` | `value ?? 'default'` |
|-------|----------|-----------------|------------------|----------------|---------------------|
| `undefined` | `true` | `true` | `false` | `false` | `'default'` |
| `null` | `true` | `true` | `true` | `false` | `'default'` |
| `''` | `true` | `false` | `false` | `true` | `''` |
| `'value'` | `false` | `false` | `false` | `false` | `'value'` |

### Real-World Example

```javascript
// Webhook processing with explicit null handling
const body = $json.body || {};

// These are the exact checks that prevent silent failures:
const validation = {isValid: true, errors: []};

// Check 1: Field exists and is not null
if (body.orderId === undefined) {
  validation.errors.push('orderId is missing');
  validation.isValid = false;
} else if (body.orderId === null) {
  validation.errors.push('orderId cannot be null');
  validation.isValid = false;
}

// Check 2: Field has actual content (not just empty)
if (body.orderId !== undefined && body.orderId !== null && body.orderId === '') {
  validation.errors.push('orderId cannot be empty');
  validation.isValid = false;
}

// After validation, safe to use:
const orderId = body.orderId;  // Guaranteed to be a real value
```

---

## Production Patterns Summary

| Pattern | Purpose | When to Add |
|---------|---------|-------------|
| Pattern 11 (Validation) | Catch bad data at entry | Tier 2+ |
| Pattern 12 (Logging) | Debug production issues | Tier 2+ |
| Pattern 13 (Error Responses) | Proper API communication | Tier 2+ |
| Pattern 14 (Null Handling) | Prevent #1 bug | Always |

**The 80/20 Rule**: For production workflows, the core logic is only 20% of the work. The other 80% is validation, error handling, and logging. See [n8n-production-readiness](../n8n-production-readiness/) for the full tier system.
