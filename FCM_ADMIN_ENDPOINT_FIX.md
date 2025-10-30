# FCM Admin Endpoint Fix

## Problem

Admin endpoint `/api/admin/send-notification` was returning **500 error** when sending notifications from the admin interface.

### Root Causes

1. **Duplicate Firebase Initialization**
   - Admin endpoint had its own Firebase initialization code (lines 79-95)
   - This caused "[DEFAULT] already exists" error
   - Conflicted with centralized `firebase-admin.ts` module

2. **Complex Direct Implementation**
   - Admin endpoint directly used `admin.messaging().sendEachForMulticast()`
   - Created complex multicast message with APNS/Android configs
   - Did not reuse the working `sendPushNotification()` helper from `firebase-admin.ts`

3. **Missing Type Safety**
   - Used `any` types in several places
   - No proper error handling for TypeScript

## Solution

### 1. Removed Duplicate Firebase Initialization

**Before:**
```typescript
import admin from 'firebase-admin';

if (typeof window === 'undefined' && !admin.apps.length) {
  // Firebase initialization code...
  admin.initializeApp({
    credential: admin.credential.cert(serviceAccount),
  });
}
```

**After:**
```typescript
import { sendPushNotification } from '@/lib/firebase-admin';
// No initialization needed - handled by centralized module
```

### 2. Simplified to Use Helper Function

**Before (Complex - 80+ lines):**
```typescript
const multicastMessage: admin.messaging.MulticastMessage = {
  notification: { title, body, imageUrl: payload.image_url },
  data: { /* complex data structure */ },
  tokens: fcmTokens,
  apns: { /* complex APNS config */ },
  android: { /* complex Android config */ },
};

const response = await admin.messaging().sendEachForMulticast(multicastMessage);
// Manual response processing...
```

**After (Simple - 15 lines):**
```typescript
const result = await sendPushNotification(
  fcmTokens,
  { title, body },
  notificationData
);

// Result already contains successCount, failureCount
```

### 3. Improved Type Safety

**Before:**
```typescript
catch (err: any) {
  debug.error = {
    message: err?.message,
    code: err?.code, // undefined property
  };
}
```

**After:**
```typescript
catch (err: unknown) {
  const error = err as Error;
  debug.error = {
    message: error?.message || 'FCM send failed',
    stackTop: typeof error?.stack === 'string' 
      ? error.stack.split('\n').slice(0, 2).join('\n') 
      : undefined,
  };
}
```

## Files Modified

### `/backend/src/app/api/admin/send-notification/route.ts`

**Changes:**
1. ✅ Import `sendPushNotification` from `@/lib/firebase-admin`
2. ✅ Remove local Firebase initialization (lines 79-95)
3. ✅ Remove Firebase initialization check (lines 107-118)
4. ✅ Simplify `sendImmediateNotification()` function:
   - Query subscribers by UUID
   - Query FCM tokens separately with locale filter
   - Use `sendPushNotification()` helper
   - Simplified from 150+ lines to ~80 lines
5. ✅ Fix TypeScript errors (`any` → `unknown` + proper casting)

## Testing Results

### Before Fix
```
POST /api/admin/send-notification 500
Error: invalid input syntax for type uuid: '1'
Error: app named '[DEFAULT]' already exists
```

### After Fix
```bash
$ curl -X POST http://localhost:3000/api/admin/send-notification \
  -H "Authorization: Bearer abc587def456ghi321-admin-lectio-divina-2024" \
  -d '{
    "title": "Test Admin Notification",
    "body": "Toto je test z admin rozhrania",
    "locale": "sk",
    "topic_id": "b8006472-7c98-4e8e-9616-d1966ee79f23"
  }'

✅ Firebase Admin SDK initialized successfully
✅ Push notification sent: 2 successful, 7 failed
POST /api/admin/send-notification 200 ✅
```

### Notification Delivery
- **2 successful** - Delivered to active devices
- **7 failed** - Expired/invalid FCM tokens (expected - need cleanup)
- **Status: 200 OK** - Admin endpoint working correctly

## Benefits

### 1. Code Reusability
- Single source of truth for Firebase initialization
- Consistent notification sending logic across endpoints
- Easier maintenance and updates

### 2. Better Error Handling
- Centralized error handling in `firebase-admin.ts`
- Proper TypeScript types
- Detailed logging of failed tokens

### 3. Consistency
- `/api/notifications/send` and `/api/admin/send-notification` now use same helper
- Same success/failure rates (2/9 tokens)
- Predictable behavior

## Environment Variables Required

```bash
# Firebase Admin SDK
FIREBASE_PROJECT_ID=lectio-divina-ef223
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-fbsvc@lectio-divina-ef223.iam.gserviceaccount.com
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"

# Admin Authentication
ADMIN_TOKENS=abc587def456ghi321-admin-lectio-divina-2024
NEXT_PUBLIC_ADMIN_TOKEN=abc587def456ghi321-admin-lectio-divina-2024
```

## Usage

### Admin Interface
1. Navigate to `/admin/notifications/new`
2. Fill form:
   - **Topic**: Select from dropdown (UUID automatically used)
   - **Title**: Notification title
   - **Body**: Notification body
   - **Locale**: Language (sk, en, etc.)
   - **Image URL** (optional)
   - **Deep linking** (optional): screen, screen_params
3. Click "Odoslať notifikáciu"
4. See success message with delivery count

### API Direct Call
```bash
curl -X POST http://localhost:3000/api/admin/send-notification \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_ADMIN_TOKEN" \
  -d '{
    "title": "Daily Reflection",
    "body": "Read today'\''s Lectio Divina",
    "locale": "sk",
    "topic_id": "UUID_FROM_notification_topics_table",
    "image_url": "https://example.com/image.jpg",
    "screen": "lectio",
    "screen_params": "{\"lectioId\":\"123\"}"
  }'
```

## Next Steps

### Optional Improvements

1. **Token Cleanup**
   - Mark invalid tokens as `is_active = false`
   - Automatically done when "Requested entity was not found" error occurs
   - Would reduce 7 failed → 0 failed in future sends

2. **Image Support Enhancement**
   - Currently image_url passed in data field
   - Could enhance `sendPushNotification()` to accept imageUrl in notification object
   - Would enable rich notifications with images

3. **Scheduled Notifications**
   - Already implemented in admin endpoint
   - Saves to `scheduled_notifications` table with `pending` status
   - Need cron job or scheduled task to send them at specified time

## Related Documentation

- [FCM Implementation Guide](./FCM_IMPLEMENTATION_SUMMARY.md)
- [FCM Quick Start](../backend/FCM_QUICKSTART.md)
- [FCM Notifications Guide](./FCM_NOTIFICATIONS_GUIDE.md)

---
**Status**: ✅ FIXED - Admin endpoint now working correctly
**Date**: 2025-01-XX
**Impact**: Admin interface notifications now functional
