# Headwind MDM Community on Docker without internet access

Notes from deploying Headwind MDM Community with the official `headwindmdm/hmdm` image on an Ubuntu 22.04 server with no direct internet access (corporate proxy with NTLM authentication). No custom image is needed: the official image works, but its entrypoint assumes internet access and has a few traps. Not affiliated with Headwind.

## What changes without internet
On start, the entrypoint downloads the WAR from `HMDM_URL` and the launcher/plugin APKs listed in its init script from h-mdm.com. Offline you provide those files yourself. The Docker daemon cannot authenticate against an NTLM proxy: download on a PC (browser, or a local forwarder such as Px) and copy to the server with `scp`.

## Files to prepare (on a PC with internet)
- Docker images `headwindmdm/hmdm:<tag>` and `postgres:17`. If the server cannot pull, transfer them with `docker save` / `docker load`.
- The WAR matching your image (see `HMDM_URL` in the `.env.example` of https://github.com/h-mdm/hmdm-docker).
- The APKs that the init script downloads. List them with:

```
docker run --rm --entrypoint grep headwindmdm/hmdm:0.1.8 h-mdm.com /opt/hmdm/templates/sql/hmdm_init.en.sql
```

In the author's case: `hmdm-6.34-os.apk`, `pager-1.02.apk`, `phoneproxy-1.02.apk`, `LauncherRestarter-1.04.apk`.

## Procedure
1. Create the folders and place the files. The WAR file name must match the last part of `HMDM_URL`, so the entrypoint skips the download:

```
mkdir -p volumes/work/cache volumes/work/files volumes/postgres volumes/hmdm-config
cp hmdm-5.39.2-os.war volumes/work/cache/
cp *.apk volumes/work/files/
```

2. Adapt `docker-compose.example.yaml` and start it. **Do not start the container before the real WAR is in `volumes/work/cache/`** (see pitfalls 1 and 2).
3. Follow the logs until you see `Server startup in`. A `SEVERE` about `hmdm.jks` is expected with `PROTOCOL=http` (unused HTTPS connector).
4. Open `http://<server>:8080`. Default login `admin` / `admin`: change it immediately.

## Pitfalls
Found and fixed on a live install. The causes were inferred by reading `docker-entrypoint.sh`.

1. **0-byte WAR.** Symptom: the WAR in `volumes/work/cache/` has 0 bytes and the panel never comes up. Cause: `wget -O` creates an empty file when the download fails, and later starts skip the download because the file exists. Fix: delete the empty file, place the real WAR, restart. Never set `FORCE_RECONFIGURE=true` offline: it deletes the cached WAR and tries to download it again.
2. **Empty `init.sql`, no applications in the panel.** Cause: the entrypoint regenerates `volumes/work/init.sql` on every start from a temporary file that only exists on the first start, so later starts overwrite it with an empty file. If the first start did not complete (for example because of pitfall 1), the database is created from an empty script. Fix on a fresh install: stop the stack, run `rm -rf volumes/postgres volumes/work/init.sql`, make sure the real WAR is in place, start again. Workaround that worked on an existing database (adapt the container names and the line numbers, which are valid for the template of image 0.1.8, and back up first):

```
docker exec <hmdm-container> sh -c "sed -n '30,184p' /opt/hmdm/templates/sql/hmdm_init.en.sql | sed 's|_HMDM_VERSION_|6.34|g; s|_HMDM_VARIANT_|os|g' > /tmp/apps_fix.sql"
docker exec -it <postgres-container> psql -U hmdm -d hmdm -c "ALTER TABLE applications DROP CONSTRAINT applications_latestversion_fkey;"
docker exec <hmdm-container> cat /tmp/apps_fix.sql | docker exec -i <postgres-container> psql -U hmdm -d hmdm
docker exec -it <postgres-container> psql -U hmdm -d hmdm -c "ALTER TABLE applications ADD CONSTRAINT applications_latestversion_fkey FOREIGN KEY (latestversion) REFERENCES applicationversions(id) ON DELETE SET NULL;"
```

3. **URLs without the port.** The entrypoint rewrites `https://h-mdm.com` to `$PROTOCOL://$BASE_DOMAIN` with no port, and the generated `base.url` in `volumes/hmdm-config/ROOT.xml` has no port either. With port 8080, enrollment cannot download the launcher. Fix: edit `base.url` in `ROOT.xml` to include the port and restart, then (adapt, back up first):

```
UPDATE applicationversions SET url = REPLACE(url, 'https://h-mdm.com/', 'http://mdm.example.com:8080/') WHERE url LIKE 'https://h-mdm.com/%';
UPDATE applicationversions SET url = REPLACE(url, 'http://mdm.example.com/', 'http://mdm.example.com:8080/') WHERE url LIKE 'http://mdm.example.com/%';
```

4. **Launcher checksum for QR provisioning.** Android expects the SHA-256 of the APK in URL-safe Base64 without padding. Compute it and store it:

```
openssl dgst -sha256 -binary volumes/work/files/hmdm-6.34-os.apk | base64 | tr '+/' '-_' | tr -d '='
UPDATE applicationversions SET apkhash='<result>' WHERE id=<launcher version id>;
```

5. **`error.internal.server` at enrollment.** In the author's database the default configuration had a NULL password. Fix: `UPDATE configurations SET password='<password>' WHERE id=1;`
6. **Android 13+ without internet during setup.** Add `"android.app.extra.PROVISIONING_ALLOW_OFFLINE": true` to the QR extras (Configurations → MDM Settings → Other QR code entries). Android 11 does not support the flag: the device needs network connectivity during setup.

## QR enrollment (example with placeholders)
Factory reset, tap 6 or 7 times on the first welcome screen (before choosing a language), and scan:

```json
{
  "android.app.extra.PROVISIONING_DEVICE_ADMIN_COMPONENT_NAME": "com.hmdm.launcher/com.hmdm.launcher.AdminReceiver",
  "android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_DOWNLOAD_LOCATION": "http://mdm.example.com:8080/files/hmdm-6.34-os.apk",
  "android.app.extra.PROVISIONING_DEVICE_ADMIN_PACKAGE_CHECKSUM": "<URL-safe base64 SHA-256 of the APK>",
  "android.app.extra.PROVISIONING_LEAVE_ALL_SYSTEM_APPS_ENABLED": true,
  "android.app.extra.PROVISIONING_SKIP_ENCRYPTION": true,
  "android.app.extra.PROVISIONING_ALLOW_OFFLINE": true,
  "android.app.extra.PROVISIONING_ADMIN_EXTRAS_BUNDLE": {
    "com.hmdm.DEVICE_ID": "device-001",
    "com.hmdm.CONFIG": "<configuration QR key from the panel>",
    "com.hmdm.GROUP": "1",
    "com.hmdm.BASE_URL": "http://mdm.example.com:8080",
    "com.hmdm.SERVER_PROJECT": ""
  }
}
```

If the launcher ends up as Device Admin instead of Device Owner, the setup wizard was probably completed before scanning. Zebra handhelds: see [docs/zebra-stagenow.md](docs/zebra-stagenow.md).

## Status
- Verified by the author: Ubuntu 22.04, image 0.1.8, WAR 5.39.2, launcher 6.34, plain HTTP, panel and QR enrollment working after the fixes above.
- Derived from reading the entrypoint, not re-run end to end: the fresh-install sequence in "Procedure" and the cause of pitfalls 1 and 2.
- Not tested: HTTPS, newer image versions, `docker compose` v1.
- Related: unofficial Premium 7.x Dockerfile, https://github.com/briancabanez-maker/hmdm-docker-premium-tomcat11
