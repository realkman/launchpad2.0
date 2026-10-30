# Security

Please report vulnerabilities privately via GitHub's "Report a vulnerability" button
(Security tab) rather than a public issue.

Known design notes:

* The Steam Web API key is stored in plain text in `settings.json`.
* "Keep me signed in" stores a random token in `session.token`; only its hash is kept in the database.
* All of your information is stored locally on your desktop.

