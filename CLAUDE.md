# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Rails 8.1.1 application for managing marketplace listings with user authentication and geocoding capabilities. Uses Tailwind CSS for styling and Action Text/Active Storage for rich content and images.

**Working Directory**: `/home/ahmed/rails/listing_items_app/rubylist/`

All commands should be run from the `rubylist` subdirectory, not the repository root.

## Key Technologies

- **Ruby**: 3.4.7
- **Rails**: 8.1.1
- **Database**: SQLite3 (development/test)
- **Styling**: Tailwind CSS
- **Authentication**: bcrypt with `has_secure_password`
- **Geocoding**: geocoder gem
- **Rich Text**: Action Text (Trix editor)
- **File Storage**: Active Storage
- **Background Jobs**: Solid Queue
- **Caching**: Solid Cache
- **Cable**: Solid Cable

## Development Commands

### Server & Development

```bash
# Start development server with all processes (web + CSS watcher)
bin/dev

# Start Rails server only
bin/rails server

# Watch Tailwind CSS changes
bin/rails tailwindcss:watch
```

### Database Operations

```bash
# Create database
bin/rails db:create

# Run migrations
bin/rails db:migrate

# Setup database (create + migrate + seed)
bin/rails db:setup

# Reset database (drop + setup)
bin/rails db:reset

# Check migration status
bin/rails db:migrate:status
```

### Testing

```bash
# Run all tests (except system tests)
bin/rails test

# Run specific test file
bin/rails test test/models/listing_test.rb

# Run specific test by line number
bin/rails test test/models/listing_test.rb:10

# Reset DB and run tests
bin/rails test:db
```

### Code Quality

```bash
# Run RuboCop linter
bin/rubocop

# Auto-correct RuboCop offenses
bin/rubocop -a

# Run Brakeman security scanner
bin/brakeman

# Run bundler-audit for gem vulnerabilities
bin/bundler-audit

# Run all CI checks (includes rubocop, brakeman, bundler-audit, tests)
bin/ci
```

### Console & Utilities

```bash
# Rails console
bin/rails console

# Database console
bin/rails dbconsole

# Generate new migration/model/controller
bin/rails generate [type] [name]
```

## Application Architecture

### Authentication Flow

Uses Rails 8's built-in authentication pattern with sessions:

- **Current**: `ActiveSupport::CurrentAttributes` for per-request user state
  - Provides `Current.user` throughout the request lifecycle
  - Delegates to `Current.session.user`
- **User**: Authenticated with `has_secure_password` (bcrypt)
  - Has many sessions and listings
  - Email normalized to lowercase and stripped
- **Session**: Database-backed user sessions (not cookie-based)
  - Belongs to user, destroyed when user is deleted
- **Authentication**: Concern included in `ApplicationController`
  - Session management and authentication helpers

### Core Models

**User** (`app/models/user.rb`)
- `has_secure_password` for authentication
- Has many sessions (dependent: destroy)
- Has many listings (dependent: destroy)
- Email normalization applied automatically

**Listing** (`app/models/listing.rb`)
- Belongs to user
- `has_rich_text :description` (Action Text for rich editing)
- `has_many_attached :images` (Active Storage for multiple images)
- Fields: name, price, location, x (latitude), y (longitude)
- **Geocoding**: Configured with `geocoded_by :location, latitude: :x, longitude: :y`
  - Auto-geocodes on save when location changes
  - Provides `near` method for proximity queries

**Session** (`app/models/session.rb`)
- Belongs to user
- Manages authentication state

### Controllers

**ListingsController** - Full CRUD for listings
- Scoped to `Current.user.listings` for multi-tenancy
- Uses strong parameters with image array support
- JSON API support included

**ExploreController** - Public listing discovery
- Root path (`/`)
- Location-based search using Geocoder
- Finds listings within 50km radius using `Listing.near`
- Note: Line 4 has typo (`:Geocoder` should be `Geocoder`)

**RegistrationsController** - User signup
**SessionsController** - Login/logout
**PasswordsController** - Password reset flow

### Routes

```ruby
root "explore#show"                    # Homepage with location search
resources :listings                    # Full CRUD for listings
resource :registrations, only: [:new, :create]
resource :session                      # Login/logout
resources :passwords, param: :token    # Password reset
```

## Database Schema

**users**
- email_address, password_digest
- timestamps

**sessions**
- user_id (foreign key)
- timestamps

**listings**
- name, price, location
- x (latitude), y (longitude) - for geocoding
- user_id (foreign key)
- timestamps
- Plus Action Text and Active Storage tables

## Styling & Frontend

- **Tailwind CSS**: Utility-first CSS framework
- **Hotwire**: Turbo + Stimulus for modern interactions
- **importmap-rails**: JavaScript without bundling
- **Propshaft**: Modern asset pipeline

Generated with `tailwind_registration` gem for styled auth forms.

## Important Patterns

### Current User Access

Always use `Current.user` in controllers, not session lookups:

```ruby
# Good
@listings = Current.user.listings

# Avoid
@listings = session[:user_id] && User.find(session[:user_id]).listings
```

### Strong Parameters for Listings

Images must be passed as an array:

```ruby
params.expect(listing: [ :name, :description, :price, :location, :x, :y, images: [] ])
```

### Geocoding Pattern

```ruby
results = Geocoder.search(location_string)
lat = results.first.coordinates[0]
lng = results.first.coordinates[1]
nearby = Listing.near([lat, lng], distance_in_km)
```

## Geocoding Setup

The Listing model uses the geocoder gem with custom latitude/longitude columns:
- `x` stores latitude
- `y` stores longitude
- Auto-geocodes when `location` field changes
- Provides `near([lat, lng], distance_km)` for proximity searches

## Security Tools

- **Brakeman**: Static security analysis
- **bundler-audit**: Checks for vulnerable gem versions
- **RuboCop Rails Omakase**: Code style and security patterns

Run security checks before committing production code.
