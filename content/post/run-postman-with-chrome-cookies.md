+++
title = 'Run Postman with Chrome Cookies'
date = 2025-03-31T12:00:00+08:00
draft = false
tags = ['Postman', 'Cookies']
categories = ['前端']
+++
When testing APIs that require authentication, manually copying cookies from Chrome to Postman can be tedious. Here's how to automatically sync Chrome cookies with Postman.

## Prerequisites

- Chrome browser
- Postman desktop application
- Chrome extension: "Cookie-Editor" or "EditThisCookie"

## Method 1: Using Cookie-Editor Extension

1. Install the "Cookie-Editor" extension from Chrome Web Store
2. Navigate to your target website and log in
3. Click the Cookie-Editor extension icon
4. Click "Export" to copy cookies in JSON format
5. In Postman, go to your request
6. Under the "Cookies" tab, click "Import"
7. Paste the copied cookies and save

## Method 2: Using Postman Interceptor ⭐️⭐️⭐️⭐️⭐️

### Step 1: Install Postman Interceptor

1. Install "Postman Interceptor" from Chrome Web Store
2. Open Postman desktop app
3. Go to Cookies > Sync Cookies
4. Set domains you want to sync cookies from
4. Click "Start Syncing"

### Step 2: Configure Interceptor

1. Click the Interceptor icon in Chrome
2. Connect to Postman desktop app
3. Select domains you want to sync cookies from

### Step 3: Use Captured Cookies

1. Your Chrome cookies will now automatically sync with Postman
2. Make API requests in Postman using the synced cookies
3. Cookies will update automatically when changed in Chrome

## Benefits

- Save time by avoiding manual cookie copying
- Always work with up-to-date cookies
- Maintain session consistency between browser and Postman

## Common Issues and Solutions

1. **Cookies Not Syncing**
   - Ensure Interceptor is properly connected
   - Check if domain is in the allowed list
   - Restart both Chrome and Postman

2. **Security Considerations**
   - Only sync cookies from trusted domains
   - Regularly clear unused cookies
   - Be cautious with sensitive cookie data

## Conclusion

Using Chrome cookies in Postman streamlines API testing workflow and ensures consistent authentication state between your browser and Postman requests.

Remember to handle cookie data securely and clear unused cookies regularly to maintain good security practices.