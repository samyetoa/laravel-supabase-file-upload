# laravel-supabase-file-upload
Complete documentation for implementing file uploads to Supabase with Laravel 11/12, Inertia.js, and Vue.js.

## Why This Guide?

This guide fills a gap in official documentation. While Laravel and Supabase have good docs separately, there's no clear guide combining them for file uploads.

## What's Included

- ✅ Step-by-step Supabase setup
- ✅ Laravel configuration (no special packages)
- ✅ Complete secureUploadImage() function
- ✅ Vue.js file input component
- ✅ Troubleshooting section
- ✅ Real working example

## Quick Start

1. Install: `composer require league/flysystem-aws-s3-v3:^3.0`
2. Configure `.env`
3. Update `config/filesystems.php`
4. Add `secureUploadImage()` method
5. Use in your controller

## Table of Contents

- [Supabase Setup](docs/supabase-setup.md)
- [Laravel Configuration](docs/laravel-config.md)
- [Implementation Guide](docs/implementation.md)
- [Troubleshooting](docs/troubleshooting.md)

## Features

- Works with Laravel 11 & 12
- No special Supabase packages needed
- 3-attempt retry logic
- Detailed error logging
- Public bucket file access
- FormData support for Vue

## License

MIT License - feel free to use and modify

## Contributing

Found an issue? Have improvements? Open a PR!

## Author

Created while solving real-world file upload problems
