# Complete Guide: Supabase File Uploads with Laravel

This guide documents how to implement file uploads to Supabase with Laravel, Inertia.js, and Vue.js.

---

## Table of Contents
1. [Composer Dependencies](#composer-dependencies)
2. [Supabase Setup](#supabase-setup)
3. [Laravel Configuration](#laravel-configuration)
4. [File Upload Implementation](#file-upload-implementation)
5. [Frontend Integration](#frontend-integration)
6. [Troubleshooting](#troubleshooting)

---

## Composer Dependencies

You **do NOT need to install any special packages** for Supabase file uploads. Laravel's built-in **Flysystem S3 driver** works with Supabase.

However, ensure you have these installed (usually pre-installed in Laravel 11/12):

```bash
composer require league/flysystem-aws-s3-v3:^3.0
```

If already installed, verify in `composer.json`:
```json
"require": {
    "league/flysystem-aws-s3-v3": "^3.0"
}
```

Then run:
```bash
composer install
```

That's it! No special Supabase packages needed.

---

## Supabase Setup

### Step 1: Create a Supabase Project
1. Go to https://app.supabase.com
2. Create a new project
3. Note your **Project URL**: `https://your-project.supabase.co`

### Step 2: Create Storage Buckets
1. Navigate to **Storage** section
2. Create two public buckets:
   - `images` - for general images
   - `products` - for product-specific files
3. Both should be **Public** (accessible without authentication)

### Step 3: Get API Credentials
1. Go to **Settings** → **API**
2. Copy these:
   - **Project URL** (your endpoint)
   - **Service Role Secret** (under "Secret keys" section)
   - **Region** (usually `eu-west-1`)

⚠️ **Important**: Use **Service Role Secret**, NOT the Anon public key!

---

## Laravel Configuration

### Step 1: Update `.env` File

Add these lines to your `.env` file (after the AWS section):

```env
# ==============================
# SUPABASE STORAGE CONFIGURATION
# ==============================
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_API_KEY=your_service_role_secret_key_here
SUPABASE_BUCKET=products
SUPABASE_REGION=eu-west-1
```

**Example:**
```env
SUPABASE_URL=https://qhalzpxchnmimmaradvc.supabase.co
SUPABASE_API_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
SUPABASE_BUCKET=products
SUPABASE_REGION=eu-west-1
```

### Step 2: Configure `config/filesystems.php`

Find the `'disks'` array and add this configuration:

```php
'disks' => [
    // ... existing disks (local, public, s3)
    
    'supabase' => [
        'driver' => 's3',
        'key' => env('SUPABASE_API_KEY'),
        'secret' => env('SUPABASE_API_KEY'),  // Same as key for Supabase
        'region' => env('SUPABASE_REGION', 'eu-west-1'),
        'bucket' => env('SUPABASE_BUCKET', 'products'),
        'endpoint' => env('SUPABASE_URL') . '/storage/v1/s3',
        'use_path_style_endpoint' => true,
        'throw' => false,
        'visibility' => 'public',
    ],
],
```

### Step 3: Clear Config Cache

```bash
php artisan config:clear
```

---

## File Upload Implementation

### Create `secureUploadImage()` Method

Add this method to your **ProductController** (or any controller handling uploads):

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Log;

class ProductController extends Controller
{
    /**
     * Securely upload an image to Supabase
     *
     * @param \Illuminate\Http\UploadedFile $file
     * @return array ['success' => bool, 'path' => string, 'url' => string, 'error' => string]
     */
    private function secureUploadImage($file)
    {
        try {
            // Validate credentials exist
            $projectUrl = env('SUPABASE_URL');
            $apiKey = env('SUPABASE_API_KEY');
            $bucket = env('SUPABASE_BUCKET', 'products');

            if (!$projectUrl || !$apiKey) {
                throw new \Exception('Missing Supabase credentials in .env');
            }

            // Generate unique filename
            $filename = time() . '_' . uniqid() . '.' . $file->getClientOriginalExtension();
            $filepath = 'main/' . $filename;

            // Read file contents
            $fileContents = file_get_contents($file->getRealPath());

            // Get storage disk
            $storage = Storage::disk('supabase');

            // Attempt 1: Upload with visibility
            Log::info('Uploading to Supabase', ['filepath' => $filepath]);
            $uploaded = $storage->put(
                $filepath,
                $fileContents,
                ['visibility' => 'public']
            );

            // Attempt 2: Upload without visibility (if first failed)
            if (!$uploaded) {
                Log::info('Retry: Uploading without explicit visibility');
                $uploaded = $storage->put($filepath, $fileContents);
            }

            // Attempt 3: Upload with ACL (if still failed)
            if (!$uploaded) {
                Log::info('Retry: Uploading with ACL');
                $uploaded = $storage->put(
                    $filepath,
                    $fileContents,
                    ['ACL' => 'public-read']
                );
            }

            // Check if all attempts failed
            if (!$uploaded) {
                throw new \Exception('Upload to Supabase S3 failed after all attempts');
            }

            // Build public URL
            $url = "{$projectUrl}/storage/v1/object/public/{$bucket}/{$filepath}";

            Log::info('Image uploaded successfully', ['url' => $url]);

            return [
                'success' => true,
                'path' => $filepath,
                'url' => $url,
            ];

        } catch (\Exception $e) {
            Log::error('SUPABASE UPLOAD FAILED', [
                'error' => $e->getMessage(),
                'file' => $file->getClientOriginalName(),
            ]);

            return [
                'success' => false,
                'error' => $e->getMessage(),
            ];
        }
    }
}
```

### Use in `store()` Method

In your controller's `store()` method, call the upload function:

```php
public function store(Request $request)
{
    try {
        // Validate request
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'product_image' => 'required|image|mimes:jpg,jpeg,png,webp|max:10240',
            'gallery_images.*' => 'nullable|image|mimes:jpg,jpeg,png,webp|max:10240',
            // ... other validations
        ]);

        // Upload main image
        if ($request->hasFile('product_image')) {
            $imageUpload = $this->secureUploadImage($request->file('product_image'));
            
            if (!$imageUpload['success']) {
                throw new \Exception('Main image upload failed: ' . $imageUpload['error']);
            }

            // Save to database
            Image::create([
                'url' => $imageUpload['url'],
                'path' => $imageUpload['path'],
            ]);
        }

        // Upload gallery images
        if ($request->hasFile('gallery_images')) {
            foreach ($request->file('gallery_images') as $galleryImage) {
                $imageUpload = $this->secureUploadImage($galleryImage);
                
                if ($imageUpload['success']) {
                    Image::create([
                        'url' => $imageUpload['url'],
                        'path' => $imageUpload['path'],
                    ]);
                }
            }
        }

        return redirect()->route('products.index')->with('success', 'Product created!');

    } catch (\Throwable $e) {
        Log::error('Product creation failed: ' . $e->getMessage());
        return back()->with('error', $e->getMessage());
    }
}
```

---

## Frontend Integration

### Vue Component (Inertia)

```vue
<template>
  <form @submit.prevent="submitForm">
    <!-- Main Product Image -->
    <div class="mb-6">
      <label class="block text-sm font-medium mb-2">Main Image</label>
      <input
        type="file"
        accept="image/*"
        @change="productImage = $event.target.files[0]"
        class="w-full border rounded px-3 py-2"
        required
      />
    </div>

    <!-- Gallery Images -->
    <div class="mb-6">
      <label class="block text-sm font-medium mb-2">Gallery Images (up to 4)</label>
      <input
        type="file"
        multiple
        accept="image/*"
        @change="galleryImages = Array.from($event.target.files).slice(0, 4)"
        class="w-full border rounded px-3 py-2"
      />
    </div>

    <!-- Other form fields -->
    <input v-model="form.name" type="text" placeholder="Product Name" required />

    <!-- Submit Button -->
    <button
      type="submit"
      :disabled="loading"
      class="bg-blue-500 text-white px-4 py-2 rounded"
    >
      {{ loading ? 'Creating...' : 'Create Product' }}
    </button>
  </form>
</template>

<script>
import { useForm } from '@inertiajs/vue3';

export default {
  data() {
    return {
      productImage: null,
      galleryImages: [],
      loading: false,
      form: useForm({
        name: '',
        // ... other fields
      }),
    };
  },
  methods: {
    async submitForm() {
      this.loading = true;

      // Create FormData for multipart upload
      const formData = new FormData();
      formData.append('name', this.form.name);
      // ... append other fields

      // Append main image
      if (this.productImage) {
        formData.append('product_image', this.productImage);
      }

      // Append gallery images
      this.galleryImages.forEach((image, index) => {
        formData.append(`gallery_images[${index}]`, image);
      });

      try {
        await this.$inertia.post(route('products.store'), formData);
      } catch (error) {
        console.error('Form validation errors:', error.response.data.errors);
      } finally {
        this.loading = false;
      }
    },
  },
};
</script>
```

---

## Verification

### Test the Upload

1. **Check Laravel logs:**
   ```bash
   tail -50 storage/logs/laravel.log
   ```
   Look for "Image uploaded successfully" message

2. **Check Supabase Dashboard:**
   - Navigate to **Storage** → **products** bucket
   - You should see new files in the `main/` folder

3. **Check Database:**
   - Your `images` table should have the uploaded image URLs

### URL Format

Uploaded files are accessible at:
```
https://your-project.supabase.co/storage/v1/object/public/products/main/filename.jpg
```

---

## Troubleshooting

### Error: "Missing Supabase credentials in .env"
- Verify `.env` has: `SUPABASE_URL`, `SUPABASE_API_KEY`, `SUPABASE_BUCKET`
- Run: `php artisan config:clear`

### Error: "Upload to Supabase S3 failed after all attempts"
- Verify `SUPABASE_API_KEY` is the **Service Role Secret** (not Anon public key)
- Check that bucket exists in Supabase Dashboard
- Verify `SUPABASE_REGION` is correct (usually `eu-west-1`)

### Files not appearing in Supabase
- Check if bucket is **Public** (not Private)
- Check Laravel logs for the actual error message
- Verify file size is within limits (10MB max)

### 403 Forbidden Error
- Bucket may be Private. Make it Public or use signed URLs for private buckets
- API key may not have permissions. Use Service Role Secret instead

---

## Summary Checklist

Before uploading to Supabase:

- [ ] Supabase project created
- [ ] Storage buckets created (public)
- [ ] `.env` configured with credentials
- [ ] `config/filesystems.php` has supabase disk
- [ ] `secureUploadImage()` method added to controller
- [ ] Image validation in form request
- [ ] `php artisan config:clear` executed
- [ ] File upload tested

---

## Key Points to Remember

1. **No special packages needed** - use League Flysystem S3 driver (pre-installed)
2. **Use Service Role Secret** - NOT the Anon public key
3. **Buckets must be Public** - for direct access without signed URLs
4. **File path structure** - organize files in folders: `main/`, `gallery/`, etc.
5. **Error handling** - always check response and log errors
6. **URL format** - `{SUPABASE_URL}/storage/v1/object/public/{bucket}/{path}`

---

## Files to Modify

| File | Change |
|------|--------|
| `.env` | Add SUPABASE_* variables |
| `config/filesystems.php` | Add supabase disk configuration |
| `app/Http/Controllers/ProductController.php` | Add secureUploadImage() method |
| `app/Http/Controllers/ProductController.php` | Call secureUploadImage() in store() |
| Vue components | Add file input fields |

---

## Next Steps

1. Use the same `secureUploadImage()` in other controllers as needed
2. For private files, implement signed URLs
3. Add image validation (dimensions, size, format)
4. Implement image cropping/optimization before upload
5. Add progress tracking for large files

