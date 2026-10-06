# ONLYOFFICE Docs

Online editors for documents, spreadsheets and presentations (ONLYOFFICE Document Server 9.4.0.1,
community edition). It has no pages of its own to use: it is the editor a Nextcloud opens files in,
through Nextcloud's **ONLYOFFICE** app.

## 1. Install and keep the JWT secret

Every request between Nextcloud and the document server is signed with a JWT secret. Harbor
generates it at install and shows it **once** in the install result (as the "password"; the name
`jwt-secret` is only a label). Copy it then.

Missed it? As root on the Harbor machine:

```sh
cat /var/lib/harbor/instances/<instance id>/secrets/provisioned-password
```

(`harbor list` / the app page show the instance id.) The value stays the same across Restart,
Update and reinstall; only a full uninstall deletes it.

## 2. Publish it under its own domain

In Harbor: **Publishing → Publish** this app on a public domain of its own (for example
`office.example.com`, an A record pointing at this machine). Nextcloud's page loads the editors
in an iframe from that address, so it must be HTTPS when Nextcloud is HTTPS. Leave basic-auth
protection off: the JWT secret protects the document server, and basic auth would break the editors
inside Nextcloud. Check it: `https://office.example.com/healthcheck` answers `true`.

## 3. Connect Nextcloud

From Harbor's terminal (Settings → Advanced access, or SSH), find Nextcloud's `app` container with
`docker ps`, then run its admin tool as `www-data` (`docker exec -u www-data <container> php occ …`):

```sh
occ app:install onlyoffice
occ config:app:set onlyoffice DocumentServerUrl --value https://office.example.com/
occ config:app:set onlyoffice jwt_secret --value <the JWT secret>
occ onlyoffice:documentserver --check
```

Or in Nextcloud: **Administration settings → ONLYOFFICE**, with the same address and the secret
under *Secret key*. The document server downloads files from Nextcloud and sends saves back to it
by Nextcloud's public address, so both apps must be able to reach each other's domain.

## What is stored where

| Storage | Holds |
|---|---|
| `data` | Document cache and converted files |
| `lib` | Editor cache and generated files |
| `db` | Internal PostgreSQL (editing sessions, change history) |
| `logs` | Document server logs |
| `fonts` | Extra fonts (`.ttf`/`.otf`); after adding some, run `documentserver-generate-allfonts.sh` in the container |

Documents themselves live in Nextcloud; this app only holds what is open or being converted.
