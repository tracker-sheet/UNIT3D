# Unit3D PHP Artisan commands reference

> **Always test these commands on a staging or test server before using them in production.**  
> Some commands can significantly alter data or disrupt service if not used correctly.




# Artisan (Laravel CLI)

> **Summary:**  
> Artisan is Laravel’s built-in command-line interface, powered by the Symfony Console component, providing dozens of pre-configured commands (such as `migrate`, `make:model`, `cache:clear`) and the ability to create custom commands, define input expectations, and schedule recurring tasks—all without writing boilerplate scripts :contentReference[oaicite:0]{index=0}.

---

## What is Artisan?

- **Artisan**  
  : The command-line interface included with Laravel, residing as the `artisan` script at your project root :contentReference[oaicite:1]{index=1}.  
- **Built on Symfony Console**  
  : Leverages Symfony’s Console component for input/output handling, command discovery, help messages, and error handling :contentReference[oaicite:2]{index=2}.  
- **Core commands**  
  : Includes tasks for database migrations (`migrate`, `migrate:rollback`), seeding (`db:seed`), cache management (`cache:clear`, `optimize`), code generation (`make:*`), and more :contentReference[oaicite:3]{index=3}.

---

## Key characteristics

- **Extensible**  
  : Custom commands can be created via `php artisan make:command`, placed in `app/Console/Commands`, and registered in `app/Console/Kernel.php` :contentReference[oaicite:4]{index=4}.  
- **Interactive I/O**  
  : Supports prompts, confirmations, hidden inputs (e.g., passwords), and progress bars for rich CLI interactions :contentReference[oaicite:5]{index=5}.  
- **Scheduling**  
  : Define recurring tasks in `Kernel.php` using a fluent scheduler (`$schedule->command(...)->daily()`) and trigger via a single cron entry (`* * * * * php artisan schedule:run`) :contentReference[oaicite:6]{index=6}.

---

## Core functionality

- **Code generation**  
  : `make:model`, `make:controller`, `make:migration`, `make:middleware`, `make:command`, etc., to scaffold boilerplate classes :contentReference[oaicite:7]{index=7}.  
- **Database management**  
  : `migrate`, `migrate:status`, `migrate:refresh`, `migrate:fresh`, `db:seed`, `db:wipe` for schema changes and data seeding :contentReference[oaicite:8]{index=8}.  
- **Cache & config**  
  : `config:cache`, `route:cache`, `view:cache`, and `optimize` to improve performance by caching compiled files :contentReference[oaicite:9]{index=9}.  
- **Queue processing**  
  : `queue:work`, `queue:retry`, `queue:failed`, `queue:monitor` to handle background jobs efficiently :contentReference[oaicite:10]{index=10}.  

---

## Custom commands

- **Create stub**  
  : `php artisan make:command CustomTask` generates a command class in `app/Console/Commands` :contentReference[oaicite:11]{index=11}.  
- **Define signature & description**  
  : Set the `$signature` (e.g., `'task:run {--force}'`) and `$description` properties in the generated class :contentReference[oaicite:12]{index=12}.  
- **Implement `handle()`**  
  : Place command logic in the `handle()` method, using injected services or direct code.  
- **Register**  
  : Add the command class to the `$commands` array in `app/Console/Kernel.php` for automatic loading :contentReference[oaicite:13]{index=13}.

---

## Scheduling commands

- **Define schedules**  
  : In `app/Console/Kernel.php`, use `$schedule->command('emails:send')->dailyAt('08:00');` :contentReference[oaicite:14]{index=14}.  
- **Single cron entry**  
  : Add `* * * * * php /path/to/artisan schedule:run >> /dev/null 2>&1` to crontab; Artisan handles timing logic :contentReference[oaicite:15]{index=15}.  
- **Monitor & catch failures**  
  : Combine with `withoutOverlapping()`, `onOneServer()`, and notification callbacks for robust scheduling :contentReference[oaicite:16]{index=16}.

---

## Further reading

- Laravel official docs: [Artisan Console](https://laravel.com/docs/artisan) :contentReference[oaicite:17]{index=17}  
- Symfony Console: [Component documentation](https://symfony.com/doc/current/components/console.html) :contentReference[oaicite:18]{index=18}  
- Task scheduling: [Laravel Scheduler](https://laravel.com/docs/scheduling) :contentReference[oaicite:19]{index=19}  










```bash
# Basic operations
php artisan about                     # Show application information
php artisan serve                     # Start development server
php artisan tinker                    # Interactive PHP shell
php artisan optimize                  # Cache framework files
php artisan down                      # Enter maintenance mode
php artisan up                        # Exit maintenance mode
php artisan migrate                   # Run pending migrations

# Achievements & authentication
php artisan achievements:load         # Load or update achievements
php artisan auth:clear-resets         # Flush expired password resets

# Automation
php artisan auto:ban_disposable_users       # Ban disposable-email users
php artisan auto:disable_inactive_users     # Disable inactive accounts
php artisan auto:deactivate_warning         # Deactivate expired warnings
php artisan auto:delete_stopped_peers       # Remove stopped peers
php artisan auto:correct_history            # Fix stuck history records
php artisan auto:bon_allocation             # Allocate bonus points
php artisan auto:group                      # Auto-update user groups
php artisan auto:cache_random_media         # Cache random media IDs
php artisan auto:email-blacklist-update     # Refresh email blacklist
php artisan auto:sync_peers                 # Sync peer counts
php artisan auto:remove_featured_torrent    # Remove expired featured torrents
php artisan auto:refund_download            # Refund downloads by seedtime
php artisan auto:sync_torrents_to_meilisearch  # Sync torrents to Meilisearch
php artisan auto:sync_people_to_meilisearch    # Sync users to Meilisearch

# Backup & cache
php artisan backup:run      # Create a new backup
php artisan backup:list     # List existing backups
php artisan backup:clean    # Clean old backups
php artisan cache:clear     # Clear application cache
php artisan set:all_cache   # Rebuild common caches

# Database & queue
php artisan db:seed         # Seed database
php artisan db:wipe         # Drop all tables
php artisan migrate:fresh   # Drop & re-run all migrations
php artisan queue:work      # Process queued jobs
php artisan queue:retry all # Retry failed jobs
php artisan queue:monitor   # Show queue statistics

# Scout (search)
php artisan scout:import             # Import models to search
php artisan scout:flush              # Flush index records
php artisan scout:index              # Create search index
php artisan scout:sync-index-settings # Sync index settings

# Troubleshooting & utilities
php artisan clear:all_cache        # Clear all caches
php artisan backup:monitor         # Check backup health
php artisan tickets:stale          # Flag stale support tickets
php artisan debugbar:clear         # Clear Debugbar storage

# Full list & help
php artisan list         # Show all available commands
php artisan help <name>  # Detailed help for a specific command
```
