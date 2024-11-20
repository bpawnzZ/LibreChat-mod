# COMPLETE IMPLEMENTATION STEPS

## 1. ENVIRONMENT SETUP

First, update the `.env` file:

```bash
# Add this line to .env file
SEARXNG_QUERY_URL=https://searxng.2damoon.xyz/search
```

---

## 2. FRONTEND MODIFICATIONS

### A. Update Input Component

**Location:** `/client/src/components/Input/`

1. Modify `TextInput.jsx`:

```javascript
// Add toggle switch import
import ToggleSwitch from './ToggleSwitch';

// Add state for search toggle
const [isSearchEnabled, setIsSearchEnabled] = useState(false);

// Add toggle switch component before the input field
<div className="relative flex items-center w-full">
  <ToggleSwitch
    isEnabled={isSearchEnabled}
    onChange={setIsSearchEnabled}
    label="Web Search"
    className="mr-2"
  />
  {/* Existing input field code */}
</div>
```

2. Create a new file: `/client/src/components/Input/ToggleSwitch.jsx`:

```javascript
const ToggleSwitch = ({ isEnabled, onChange, label, className }) => {
  return (
    <div className={`flex items-center ${className}`}>
      <label className="relative inline-flex items-center cursor-pointer">
        <input
          type="checkbox"
          checked={isEnabled}
          onChange={(e) => onChange(e.target.checked)}
          className="sr-only peer"
        />
        <div className="w-11 h-6 bg-gray-200 peer-focus:outline-none peer-focus:ring-4 peer-focus:ring-blue-300 dark:peer-focus:ring-blue-800 rounded-full peer dark:bg-gray-700 peer-checked:bg-blue-600 peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-gray-300 after:border after:rounded-full after:h-5 after:w-5 after:transition-all dark:border-gray-600"></div>
        <span className="ml-3 text-sm font-medium text-gray-900 dark:text-gray-300">
          {label}
        </span>
      </label>
    </div>
  );
};

export default ToggleSwitch;
```

---

## 3. BACKEND MODIFICATIONS

### A. Create SearXNG Service

**Location:** `/api/app/clients/tools/`

1. Create a new file: `searxng.js`:

```javascript
const axios = require('axios');
require('dotenv').config();

class SearXNGService {
  constructor() {
    // Use environment variable for base URL
    if (!process.env.SEARXNG_QUERY_URL) {
      throw new Error('SEARXNG_QUERY_URL environment variable is not set');
    }
    this.baseURL = process.env.SEARXNG_QUERY_URL;
  }

  async search(query) {
    try {
      const response = await axios.get(this.baseURL, {
        params: {
          q: query,
          engines: 'duckduckgo,google',
          format: 'json',
          count: 15
        }
      });

      return this.processResults(response.data.results);
    } catch (error) {
      console.error('SearXNG search error:', error);
      throw error;
    }
  }

  processResults(results) {
    // Process and summarize results
    const processed = results.map(result => ({
      title: result.title,
      link: result.url,
      snippet: result.content
    }));

    return processed;
  }
}

module.exports = SearXNGService;
```

### B. Update API Routes

**Location:** `/api/app/routes/`

1. Create a new file: `search.js`:

```javascript
const express = require('express');
const router = express.Router();
const SearXNGService = require('../clients/tools/searxng');

const searxng = new SearXNGService();

router.post('/search', async (req, res) => {
  try {
    const { query } = req.body;
    const results = await searxng.search(query);
    res.json(results);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

2. Update `/api/app/server.js`:

```javascript
// Add new route
const searchRoutes = require('./routes/search');
app.use('/api/search', searchRoutes);
```

---

## 4. INTEGRATION LOGIC

### A. Update Message Processing

**Location:** `/api/app/`

1. Modify `handleMessage.js`:

```javascript
const SearXNGService = require('./clients/tools/searxng');
const searxng = new SearXNGService();

// Add to message processing logic
async function handleMessage(message, isSearchEnabled) {
  if (isSearchEnabled) {
    try {
      const searchResults = await searxng.search(message);
      // Process and summarize search results
      const summary = await summarizeResults(searchResults);
      return {
        text: message,
        searchResults: summary
      };
    } catch (error) {
      console.error('Search error:', error);
      return {
        text: message,
        error: 'Search failed'
      };
    }
  }
  return { text: message };
}
```

---

## 5. FRONTEND STATE MANAGEMENT

### A. Add Search State to Store

**Location:** `/client/src/store/`

1. Create a new file: `searchStore.js`:

```javascript
import create from 'zustand';

const useSearchStore = create((set) => ({
  isSearchEnabled: false,
  setSearchEnabled: (enabled) => set({ isSearchEnabled: enabled }),
}));

export default useSearchStore;
```

---

## 6. ERROR HANDLING FOR ENVIRONMENT VARIABLES

### A. Create Environment Variable Validation

**Location:** `/api/app/config/`

1. Create a new file: `envValidation.js`:

```javascript
function validateEnvironmentVariables() {
  const requiredVars = ['SEARXNG_QUERY_URL'];
  
  for (const variable of requiredVars) {
    if (!process.env[variable]) {
      throw new Error(`Required environment variable ${variable} is not set`);
    }
  }
}

module.exports = validateEnvironmentVariables;
```

2. Update `/api/app/server.js`:

```javascript
const validateEnvironmentVariables = require('./config/envValidation');

// Add at startup
validateEnvironmentVariables();
```

---

## 7. TESTING

### Create Test Files

A. Create `/api/app/tests/searxng.test.js`:

```javascript
const SearXNGService = require('../clients/tools/searxng');

describe('SearXNG Service', () => {
  beforeAll(() => {
    process.env.SEARXNG_QUERY_URL = 'https://searxng.2damoon.xyz/search';
  });

  test('should initialize with correct URL', () => {
    const service = new SearXNGService();
    expect(service.baseURL).toBe(process.env.SEARXNG_QUERY_URL);
  });

  test('should throw error if SEARXNG_QUERY_URL is not set', () => {
    delete process.env.SEARXNG_QUERY_URL;
    expect(() => new SearXNGService()).toThrow();
  });
});
```

---

## 8. DOCUMENTATION UPDATE

Update `README.md` to include:

```markdown
## Environment Variables

The following environment variables are required:

- `SEARXNG_QUERY_URL`: The base URL for SearXNG search queries (e.g., https://searxng.2damoon.xyz/search)

Make sure to set these in your .env file before running the application.
```

---

## Implementation Notes

1. The `SEARXNG_QUERY_URL` environment variable is now required.
2. Error handling is in place if the variable is not set.
3. The service will fail gracefully with appropriate error messages.
4. Tests verify the environment variable handling.
5. Documentation is updated to reflect the requirements.

---

## Additional Considerations

1. Consider adding URL validation for the `SEARXNG_QUERY_URL`.
2. Add an environment variable for the number of results to fetch.
3. Consider adding backup URLs if the primary fails.
4. Implement environment variable caching.
5. Add an environment variable for timeout settings.

---

This implementation provides a robust way to configure the SearXNG service while maintaining security and flexibility. The code is well-structured and includes proper error handling and validation.
