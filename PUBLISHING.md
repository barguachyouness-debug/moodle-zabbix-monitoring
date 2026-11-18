# Publish this project to GitHub

The repository package is prepared locally. Review it before making the repository public.

## 1. Review the files

- Confirm the examples contain placeholders, not real passwords.
- Add only sanitized Zabbix screenshots to `screenshots/`.
- Check that no Moodle backups, database dumps, private keys, `config.php`, real `mysql.conf`, or logs are present.
- Choose a repository license before publishing if you want others to reuse the material.

A final basic check from the project directory:

```bash
find . -type f \( -name '*.key' -o -name '*.pem' -o -name '*.sql' -o -name '*.sql.gz' -o -name 'config.php' -o -name 'mysql.conf' \) -print

git status --short
```

The `find` command should not return any secret-bearing or data files. Review staged files manually even if it returns nothing.

## 2. Create the GitHub repository

On GitHub, create a new repository named:

`moodle-zabbix-monitoring`

For the first upload, create it empty (do not initialize it with a README, license, or `.gitignore`, because those files are already in this package).

## 3. Push from CachyOS

Extract the ZIP archive, open a terminal in the extracted `moodle-zabbix-monitoring` directory, and run:

```bash
git init -b main
git add .
git status --short
```

Review the staged files. If everything is safe, commit:

```bash
git commit -m "Initial Moodle Zabbix monitoring lab documentation"
```

Then connect the empty GitHub repository. Replace `YOUR_GITHUB_USERNAME` with your actual GitHub username:

```bash
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/moodle-zabbix-monitoring.git
git push -u origin main
```

GitHub may ask you to authenticate through a browser or a configured credential manager. Do not put a GitHub token directly in the remote URL or commit it to the repository.

## 4. After publishing

- Add the repository to the Featured section of your LinkedIn profile.
- Add the project under LinkedIn's Projects section with a concise description and the repository URL.
- Publish a short post describing the lab objective, architecture, verified metrics, and one or two troubleshooting lessons.
