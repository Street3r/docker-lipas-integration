# Security policy

## Reporting a vulnerability

Please do not report security problems in public issues.

Report them privately via **Security → Report a vulnerability** in this repository (GitHub private vulnerability reporting). Include what you found, how to reproduce it and what it affects.

You will get a reply as soon as possible. Fixes are made on the `main` branch; there are no separately maintained older versions.

## Scope

The Docker Compose stack and nginx configuration in this repository. Keep `.env`, `secrets/` and `certs/` out of git; they are listed in `.gitignore`.
