# tf_wp_cleanup

Run it as: `tf_wp_cleanup`

A single-file Python tool to **find, clean and recover hacked WordPress sites on shared hosting** (cPanel, SSH or the cPanel Terminal). It needs no installation and no extra Python packages.

> **Read this first.** No scanner catches everything. This tool is careful by design (dry run by default, nothing is deleted, everything it changes can be undone), but you must **take a full backup before using `--fix` or `--recover`**. Use it at your own risk.

## What it does

| Problem | What the tool does |
|---|---|
| Modified or missing WordPress core files | Compares every file with the official checksums from wordpress.org and restores the originals from the official ZIP |
| Infected plugins from wordpress.org | Same check per plugin and version; restores modified or missing files |
| Backdoors and extra files | Finds PHP files inside `wp-content/uploads`, PHP hidden in images, unknown PHP in `wp-admin` / `wp-includes`, known web shells, `eval(base64_decode(...))` and similar patterns |
| Chinese / Japanese / Korean spam text | Flags files that contain CJK characters (turn off with `--no-cjk`) |
| Malicious `.htaccess`, `.user.ini`, `auto_prepend_file` | Flags and strips the loader lines; can rewrite `.htaccess` with the standard WordPress rules |
| Site down (500 error, "critical error") | Reads your error logs and tells you which file/plugin/theme crashed and what to run |
| Broken permissions (777/666 cause 500 errors) | Resets folders to 755 and files to 644 |
| Lost administrator rights | Checks the admin role and user list, repairs the role |
| Missing or infected `wp-config.php` | Rebuilds a clean one from the old database settings, with new security keys |

**What it does NOT do:** it does not clean your **database** (spam inside posts, options or users), does not repair infected **themes or premium plugins** (they have no official checksums, so they are only reported), and does not remove the original security hole. Update everything, remove nulled/pirated plugins and change all passwords.

## Safety design

- **Dry run by default.** A plain run only reports. Nothing changes until you add `--fix`.
- **Nothing is deleted.** Suspicious files are *moved* to a private quarantine folder; replaced files are backed up first. Put anything back with `--undo-file`. The only command that deletes is `--purge-quarantine`, and it asks first.
- **`wp-config.php` is never moved automatically.**
- **Private output folders.** Reports and quarantine are created next to the script with permission `700` and a deny-all `.htaccess`.
- **No personal data is collected or sent anywhere.** The only network requests go to `api.wordpress.org` and `downloads.wordpress.org` for official checksums and ZIPs (use `--no-network` to skip them).

## Requirements

- Python 3.6 or newer (`python3 --version`)
- Shell access: SSH, or **cPanel > Terminal** (Advanced). If you don't see Terminal, ask your host to enable SSH.
- Outbound HTTPS to wordpress.org (for the official checksums)
- PHP command line (`php`) - only needed for `--check-admin` and `--fix-admin`

## Install

Put the script in a folder **outside your websites** (never inside `public_html`):

```bash
mkdir -p ~/tf_tools/tf_cleanup_wp
cd ~/tf_tools/tf_cleanup_wp
# either upload tf_wp_cleanup.py with cPanel File Manager, or:
curl -O https://raw.githubusercontent.com/YOUR_GITHUB_USER/YOUR_REPO/main/tf_wp_cleanup.py
python3 tf_wp_cleanup.py --version
```

Then create the `tf_wp_cleanup` command as shown in *Quick start* below.

Everything the tool writes goes **next to the script**:

```
~/tf_tools/tf_cleanup_wp/
├── tf_wp_cleanup.py
├── tf_wp_reports/       run logs (run_<date>.log) and findings reports (report_<date>.txt)
└── tf_wp_quarantine/    one folder per run: moved files + backups of replaced files
```

The command works through a link, but the reports and quarantine always stay next to the real script. Use `--reports-dir` and `--quarantine-dir` to choose other locations.

## Quick start for non-technical users

Run the tool **from the site's own folder** (the one that contains `wp-config.php`, `wp-admin` and `wp-includes`). It only looks at the folder you are standing in and everything below it.

**One-time setup:** create the `tf_wp_cleanup` command (copy and paste, one block):

```bash
chmod +x ~/tf_tools/tf_cleanup_wp/tf_wp_cleanup.py
mkdir -p ~/bin
ln -sf ~/tf_tools/tf_cleanup_wp/tf_wp_cleanup.py ~/bin/tf_wp_cleanup
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
tf_wp_cleanup --version
```

From now on the tool is simply **`tf_wp_cleanup`**. If the last line prints the version, you're done. (If the command is not found, you can always run the same thing with the full form: `python3 ~/tf_tools/tf_cleanup_wp/tf_wp_cleanup.py`.)

**Every time you check a site:**

```bash
cd ~/public_html/yoursite     # 1. go to the site folder
tf_wp_cleanup                       # 2. scan only - changes nothing
                              # 3. read the result (see "Reading the result" below)
tf_wp_cleanup --fix                 # 4. apply the safe fixes (backup first!)
tf_wp_cleanup                       # 5. scan again: you want "Nothing suspicious found."
tf_wp_cleanup --diagnose            # 6. if the site shows an error, find out why
```

Check every site **after any suspicious event** (strange pages, Google warnings, hosting abuse emails, a plugin you didn't install) and at least **once a month**.

**Several sites in one account?** Scan them all, read-only:

```bash
for d in ~/public_html/*/; do (cd "$d" && [ -f wp-config.php ] && tf_wp_cleanup); done
```

**Reading the result:**

- `[HIGH]` - almost certainly bad or modified. Deal with these first.
- `[MED]` - suspicious, needs a human look (some are normal, e.g. a caching plugin's files).
- `[INFO]` - for your information.
- The arrow at the end tells you what `--fix` will do: `restore` (replace with the official file), `quarantine` (move away), `review` (the tool will not touch it), `aggr` (only with `--aggressive`).

## One-command recovery

```bash
cd ~/public_html/yoursite
tf_wp_cleanup --recover
```

`--recover` = `--fix` + `--htaccess` + `--fix-perms` + `--diagnose`. Add `--aggressive` to also quarantine unknown PHP files in plugins and in the site root: `tf_wp_cleanup --recover --aggressive`.

## Commands

**Scanning and fixing**

| Command | What it does |
|---|---|
| *(no options)* | Scan, report only |
| `--fix` | Apply safe fixes: restore core/plugin files, quarantine PHP in uploads and unknown PHP in core folders, strip `auto_prepend_file` lines |
| `--aggressive` | With `--fix`: also quarantine unknown PHP in plugins, unknown PHP in the site root, and files with high-confidence malware signatures |
| `--recover` | Everything needed to get a site back online (see above) |
| `--days N` | Also list PHP files changed in the last N days |
| `--no-cjk` | Don't flag Chinese/Japanese/Korean characters |
| `--no-network` | Skip the official checksum checks |
| `--exclude PATH` | Ignore a folder (e.g. an old backup): `--exclude bkp` |

**Recovery tools** (these act immediately and can be reversed)

| Command | What it does |
|---|---|
| `--diagnose` | Read error logs, explain the crash, suggest the exact command |
| `--debug on` / `--debug off` | Turn WordPress error logging on/off (`wp-content/debug.log`); reproduce the problem, then run `--diagnose` |
| `--htaccess` | Rewrite `.htaccess` with the standard WordPress block. Subfolder site: `--rewrite-base /blog/`. Keep cPanel's PHP-version lines: `--keep-php-handler` |
| `--fix-perms` | Folders 755, files 644, `wp-config.php` 600 |
| `--disable-all-plugins` / `--enable-all-plugins` | Rename `wp-content/plugins` to `plugins.disabled` and back |
| `--disable-plugin SLUG` / `--disable-theme SLUG` | Rename one plugin/theme folder to `SLUG.disabled` |
| `--rebuild-config` | Rebuild a clean `wp-config.php` (keeps DB settings, new security keys). If the old file is gone: `--db-name --db-user --db-password --db-host --table-prefix` |
| `--check-admin` | Read-only: administrator role, table prefix, user list (new accounts flagged) |
| `--fix-admin USER` | Make this user (login or ID) an administrator and repair a broken role |
| `--strip-conf FILE` | Remove `auto_prepend_file` / `auto_append_file` lines from one `.user.ini`, `php.ini` or `.htaccess` |
| `--quarantine-file FILE` | Move one file to quarantine |

**Undo and cleanup**

| Command | What it does |
|---|---|
| `--list-quarantine` | Show everything that was moved or replaced |
| `--undo-file /full/path/file` | Put one file back |
| `--undo-all [RUN]` | Put back a whole run (asks first - this also brings the malware back) |
| `--purge-quarantine DAYS` | **Permanently delete** quarantine runs older than DAYS (asks first; `--yes` skips the question, for cron) |

## Typical situations

**The site shows a 500 error or "critical error"**
```bash
tf_wp_cleanup --diagnose                 # names the file / plugin / theme
tf_wp_cleanup --disable-plugin SLUG      # if it points to a plugin (or --disable-theme)
tf_wp_cleanup --htaccess                 # if the log shows nothing
tf_wp_cleanup --fix-perms
```
Also check the PHP version in cPanel > **MultiPHP Manager**: old plugins often crash on PHP 8.

**I can't edit pages or my admin has no permissions**
```bash
tf_wp_cleanup --check-admin
tf_wp_cleanup --fix-admin YOUR_USERNAME
```
A **403 Forbidden** is a different problem (server rules, a security plugin, ModSecurity or a leftover `.htaccess`), not a WordPress permission.

**`wp-config.php` is missing or infected**
```bash
tf_wp_cleanup --rebuild-config
```
Check the values it prints (especially `WP_HOME`, `WP_SITEURL` and `DB_HOST`) and review the list of settings it did not copy.

**I need something back that the tool moved**
```bash
tf_wp_cleanup --list-quarantine
tf_wp_cleanup --undo-file /home/YOUR_USER/public_html/yoursite/path/to/file.php
```

## Scheduling (optional)

In cPanel > **Cron Jobs**. Cron does not load your shortcuts, so use the full `python3 /home/YOUR_USER/...` form with your real home path:

```
# weekly read-only scan of one site (reports are saved in tf_wp_reports/)
0 3 * * 1  cd /home/YOUR_USER/public_html/yoursite && python3 /home/YOUR_USER/tf_tools/tf_cleanup_wp/tf_wp_cleanup.py >/dev/null 2>&1

# daily cleanup of quarantine runs older than 30 days
0 4 * * *  python3 /home/YOUR_USER/tf_tools/tf_cleanup_wp/tf_wp_cleanup.py --purge-quarantine 30 --yes >/dev/null 2>&1
```

Don't schedule `--fix` or `--fix-admin` unattended.

## After a cleanup - the checklist the tool cannot do for you

1. Change **all** passwords: hosting panel, FTP/SSH, database, every WordPress admin.
2. Update WordPress, plugins and themes; delete anything you don't use; never use nulled/pirated plugins.
3. Check **Users** for administrator accounts you didn't create (`--check-admin` flags recent ones).
4. Check cPanel **Cron Jobs** and other folders/sites on the same account - one infected site often infects its neighbours.
5. Look for old backups or copies of the site inside `public_html` (such as `bkp/`): they are often infected and reachable from the web. Delete them or move them out.
6. Ask your host whether a server-side malware scanner is available.
7. Keep regular off-site backups.

## Upgrading / moving the tool

If you move the tool to a new folder, move its output folders with it, otherwise `--list-quarantine` and `--undo-file` will not see older runs:

```bash
mv OLD_FOLDER/tf_wp_quarantine/* ~/tf_tools/tf_cleanup_wp/tf_wp_quarantine/
mv OLD_FOLDER/tf_wp_reports/*    ~/tf_tools/tf_cleanup_wp/tf_wp_reports/
```

Then re-run the `ln -sf ...` line from *Quick start* so the `tf_wp_cleanup` command points at the new location.

## Limitations

- Detection is pattern-based: expect some false positives and some misses.
- Themes, premium plugins and custom code can only be scanned, not verified. Reinstall them from a clean copy.
- Heavily obfuscated malware may need a professional clean-up.
- Tested on Linux shared hosting; other environments may need adjustments.

## License and disclaimer

Add a `LICENSE` file of your choice (MIT is common for tools like this). The software is provided "as is", without warranty of any kind. You are responsible for backups and for checking the result.

## Contributing

Issues and pull requests are welcome. If you report a false positive, please include the matching snippet **with any passwords, keys and personal data removed**.
