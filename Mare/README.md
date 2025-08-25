# Self-hosting a Mare Sync service

This assumes you are running some kind of Linux distribution using Systemd for service management. The Docker files available are NOT used. If you prefer to run Docker, do not follow this guide.

The examples here will assume a single server will be responsible for running both the main server as well as the file server. This is probably OK for services with less than 1000 concurrent users.

## Some notes about Mare

* Mare's cache file format is not compatible with Loporrit, due to using a different file compression format.
* The databases are not compatible.
* The clients are not compatible with each-others' servers.
* Mare requires that you accept websocket connections directly on the domain specified in the `wss://` server URL.
* Mare does not support the use of cold storage on sharded file-servers.
* There are some small configuration differences, and additional endpoints must be configured.
* Registering an account on a Mare server is only possible via Discord, and requires lodestone authentication. You will not be able to easily test the service until you get the Discord bot up and running to register an account.
* Thus, setting up a Discord bot (and ideally also Discord OAuth application) is required without extensive code changes. Discord bot service for Loporrit is only useful for sending admin messages and handling profile reports.

# Pre-requisites

## 0. Domain and Server specs

Firstly, you should have a server available with adequate specs

| Users | Peak Online | SSD | RAM | Net speed | Monthly Transfer |
|-|-|-|-|-|-|
| 1-50  | 10-20 | 10-50 GB | 1 GB+ | 100 Mbit+ | 1 - 10 TB |
| 10-1000 | 100-200 | 100-500 GB | 2 GB+ | 100 - 500 MBit+ | 10 - 50 TB |
| 1000+ | 200+ | 300-1000 GB | 4-8 GB+ | 500 - 1000 MBit+ | 50 TB + |
| 5000+ | 1000+ | 1000 GB+ | 8-16 GB+ | 1000 MBit+ | 150 TB + |

Additional disk space and network bandwidth will result in a better experience for users, but too much will increase RAM requirements for no little benefit. Exact requirements will vary depending on the differences in user behavior.

It may be possible to take advantage of fast HDD arrays as *cold storage*, or use a hybrid array using SSDs as a caching layer, but it may be better to simply rely on users needing to re-upload files more often than to try to make use of slower HDDs.

You should *not* run the service through CloudFlare, as it violates their terms of service and is very likely to show suspiciously high network transfer. They also restrict the number of concurrent websocket connections for free users.

You should also have a domain name that you can rely on to keep for many years. This will be used to connect to your server and it is not possible to change it later.

For example, if you register the domain `example.sync`, you will connect to your service using: `wss://example.sync`

**Mare Warning:** in particular, you may wish to use a sub-domain, not your primary domain, so that you are free to use Cloudflare or another CDN for your primary website.

## 1. Install required software: nginx, Redis, and PostgreSQL

Examples here will assume your server is running Debian:

```sh
apt install nginx postgresql redis
systemctl start nginx postgresql redis
```

It is possible to use a webserver other than nginx, however the sample configuration here is for nginx, and LoporritSync has an optimized file transfer mode which depends on nginx.

## 1a. Set up a website and install certbot

You should set up a website initially, as the server uses web technologies for its connectivity:

```sh
mkdir /var/www/example.sync/
echo "Hello world" > /var/www/example.sync/index.html
```

Create a file at `/etc/nginx/sites-enabled/example.sync.conf` with a basic configuration, e.g.:
```nginx
# include /etc/nginx/mare_upstream.conf;

server {
	listen 80;

	server_name example.sync;
	root /var/www/example.sync/;

	# include /etc/nginx/mare_server.conf;
	# include /etc/nginx/mare_files.conf;
	# include /etc/nginx/mare_cdn.conf;
}
```

( The `include` directives will be commented out later )

* Run `nginx -t` to validate your configuration.
* Run `nginx -s reload` (or `systemctl reload nginx`) to reload your configuration changes.

This configuration will be expanded later on to allow connecting to the Loporrit Sync services through it.

## 1b. Set up SSL using certbot

It is strongly recommended that you enable HTTPS on your website for user privacy.

Before doing this, ensure that nginx is serving your website correctly over http:

```sh
curl -i http://example.sync/
```

If you receive an nginx default webpage, rather than the "hello world" page created above, you need to go back and fix your nginx site config first, otherwise certbot will mess things up.

Once you confirm that the site config is correct, you can use this to have certbot enable HTTPS automagically for you.

```sh
apt install certbot
certbot --nginx -d example.sync
```

You do not have to use cerbot or its nginx module, but it is the simplest way to proceed.

After this command succeeds, certbot will have modified `/etc/nginx/sites-enabled/example.sync.conf`, and you should be able to reach your website over https e.g. `https://example.sync`

LetsEncrypt Privacy Policy: https://letsencrypt.org/privacy/

## 1c. Firewall, TCP tuning and rate-limiting

You probably look in to configuring a firewall to lock down the server and help mitigate connection flooding attacks.

For good download speeds for high latency users, also look in to optimizing the TCP buffer sizes via sysctl.

I also recommend having some kind of rate limiting policy configured in nginx, as well.

## 2. Install .NET and ASP.NET runtime from Microsoft's custom repository following the instructions here:

https://learn.microsoft.com/en-us/dotnet/core/install/linux-debian?tabs=dotnet9

At least .NET 9 is required, but .NET 10 should be used as soon as its released and stable.

## 3. Create a database and user account

```sh
adduser mare
sudo -u postgres psql
CREATE USER mare;
CREATE DATABASE mare WITH OWNER mare;
\q
```

Note: `sudo -u postgres psql` is the command needed to connect to postgres as superuser.

Note: `MareSynchronosServer` will initialize the database when it is started for the first time.

## 4. Create a cache directory

This should be somewhere there is adequate space. Take note of the space available for later.

```sh
mkdir /mnt/marecache/
for x in 0 1 2 3 4 5 6 7 8 9 A B C D E F; do mkdir /mnt/marecache/$x; done
chown -R mare:mare /mnt/marecache/
```

Check available disk space after:

```
df -h /mnt/marecache/
```

# Building and uploading

You should simply be able to compile the server software on your PC, and then copy the contents of each `bin/Release/net9.0/` directory on to the server.

It is OK to upload the contents of multiple server programs in to the same directory, overwriting the files each time. We will use the directory `/opt/mare-server/` to hold the files for all three required server programs.

For example:

```sh
cd MareSynchronosServer
dotnet build --configuration Release
scp -rp MareSynchronosServer/bin/Release/net9.0/ root@example.sync:/opt/mare-server/
scp -rp MareSynchronosAuthService/bin/Release/net9.0/ root@example.sync:/opt/mare-server/
scp -rp MareSynchronosStaticFilesServer/bin/Release/net9.0/ root@example.sync:/opt/mare-server/
scp -rp MareSynchronosServices/bin/Release/net9.0/ root@example.sync:/opt/mare-server/
```

**Mare Warning:** AuthService will, by default, copy an `appsettings.json` file in to the build output folder. This issue is addressed in suggested code patches at the bottom of this document.

# Configuration

The server would normally be configured using the file `/opt/mare-server/appsettings.json`.

We take advantage of the `ASPNETCORE_ENVIRONMENT` feature to use named configuration files per-service, read from the same directory, instead:

* `/opt/mare-server/appsettings.MareAuth.json`
* `/opt/mare-server/appsettings.MareServer.json`
* `/opt/mare-server/appsettings.MareFiles.json`
* `/opt/mare-server/appsettings.MareServices.json` (discord bot)

The options in each of these will be applied in addition to the base configuration from `appsettings.json`.

Copy the initial configuration files in `/opt/mare-server/` on your server from these examples:

* https://github.com/loporrit/SelfHosting/tree/main/Mare/ExampleConfig/MareConfig

**IMPORTANT: You will need to, at minimum, edit the following options:**

- **All appsetting.json files**:
  - **Jwt**: This should be a randomly generated / face-rolled string, unique to be server. Keep it secret. It authenticates connections between your server processes.
  - **CdnFullUrl**: Should be the URL to your website, e.g `https://example.sync/`

- **appsettings.MareAuth.json**:
  - **PublicOAuthBaseUrl**: If you want to use oauth: `https://example.sync/oauth/` with the trailing slash
  - **DiscordOAuthClientSecret**: Discord OAuth app client secret...
  - **DiscordOAuthClientId**: And the Disord OAuth app client ID

- **appsettings.MareFiles.json**:
  - **CacheDirectory**: Make sure this matches the cache directory location you created before
  - **CacheSizeHardLimitInGiB**: This should be less than the disk space available. e.g. if you have 150GB of space available, consider setting this to 120.

- **appsettings.MareServices.json**:
  - **DiscordBotToken**: The secret token for your discord bot
  - **DiscordChannelForMessages**: Discord channel ID that the bot will post its account management/registration embed to. Make it public but don't let people send mesages.
  - **DiscordChannelForCommands**: Discord channel ID for running bot commands. Should be private for admins and the bot only.
  - **DiscordRoleRegistered**: Optional, but a role ID for the bot to assign to users who register :)
  - **VanityRoles**: If this is left empty anyone can assign vanity IDs via the bot. Its a json dict of `{"RoleID":"Role Name"}`.

**Mare Warning:** The example configuratons provided with the official Mare docker examples do not assign a port number for AuthService, causing it to conflict with the Discord bot by default. This issue is addressed in suggested code patches at the bottom of this document.

You should be able to test the configuration with the following command. This first execution is also important to initialize the database:

```sh
mkdir /run/mare-server
chown mare:mare /run/mare-server
sudo -u mare ASPNETCORE_ENVIRONMENT=MareServer ./MareSynchronosServer
```

If there are any errors, try to correct them. If there are no errors, hit Ctrl+C to terminate the server.

# Install the services to start on boot using Systemd

This is the recommended way to set up the Loporrit Sync server to start on boot.

Create the following systemd unit files in `/etc/systemd/system/` on your server from these examples:

* https://github.com/loporrit/SelfHosting/tree/main/ExampleConfig/Systemd

And run the following command to set them to start on boot:

```sh
systemctl enable mare-server mare-auth mare-files
```

( For now, skip `mare-services.service` which is the discord bot, as it requires additional setup to create a bot. )

Once the service files are installed, run `systemctl daemon-reload` to reload the configuration files.

If that succeeds, run `systemctl start mare.target` to manually start the services.

## The inevitable debugging

Something will probably go wrong at this point and need to be diagnosed.

You can use the following commands to investigate the status of each service:
* `systemctl status mare-server`
* `systemctl status mare-auth`
* `systemctl status mare-files`

You can also open a second terminal to investigate the system log file, which is where errors will be reported:

* `journalctl -n 1000` - Output the last 1000 lines of the system log file
* `journalctl -f` - Live feed of the system log file (tip: keep this open a second terminal while you're setting things up)

You may also also need to know the following commands later:

* `tail -n 100 /var/log/nginx/access.log` - Last 100 requests to your website
* `tail -n 100 /var/log/nginx/error.log` - Last 100 errors

Once everything is confirmed to be working, you may wish to `reboot` at this point to validate that the services come up correctly.

# Set up services to be available via nginx

The final step is to enable connecting to the server processes via your webserver.

Copy the following files to `/etc/nginx/` on your server from these examples:

* https://github.com/loporrit/SelfHosting/tree/main/Mare/ExampleConfig/nginx

Finally, edit the website config file `/etc/nginx/sites-enabled/example.sync.conf` created previously, and **un-comment the four previously added `include` directives**, e.g.:

```nginx
include /etc/nginx/mare_upstream.conf;

server {
	listen 80;

	server_name example.sync;
	root /var/www/example.sync/;

	include /etc/nginx/mare_server.conf;
	include /etc/nginx/mare_files.conf;
	include /etc/nginx/mare_cdn.conf;
}
```

( Note that this file will have been edited slightly by certbot, if you set that up previously )

* Run `nginx -t` to validate your configuration.
* Run `nginx -s reload` (or `systemctl reload nginx`) to reload your configuration changes.

At this point it should be possible to connect to the service by setting the service url to `wss://example.sync`.

Check `/xllog` in-game, and the server log files documented above to diagnose any issues.

# Setting up Discord

You need a bot which can post in channels (give it Send Messages and Use Application Commands permissions -- also Assign Roles and Kick Members if you want either of those things). You also need an oauth thingy if you want oauth to work. The redirect URL should be `https://example.sync/oauth/discordCallback`

... Good luck!

# Recommended Code Patches

## 1. Remove MareSynchronosServerTest

```diff
--- a/MareSynchronosServer/MareSynchronosServer.sln
+++ b/MareSynchronosServer/MareSynchronosServer.sln
@@ -7,8 +7,6 @@ Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "MareSynchronosServer", "Mar
 EndProject
 Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "MareSynchronos.API", "..\MareAPI\MareSynchronosAPI\MareSynchronos.API.csproj", "{326BFB1B-5571-47A6-8513-1FFDB32D53B0}"
 EndProject
-Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "MareSynchronosServerTest", "MareSynchronosServerTest\MareSynchronosServerTest.csproj", "{25A82A2A-35C2-4EE0-A0E8-DFDD77978DDA}"
-EndProject
 Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "MareSynchronosShared", "MareSynchronosShared\MareSynchronosShared.csproj", "{67B1461D-E215-4BA8-A64D-E1836724D5E6}"
 EndProject
 Project("{9A19103F-16F7-4668-BE54-9A1E7A4F7556}") = "MareSynchronosStaticFilesServer", "MareSynchronosStaticFilesServer\MareSynchronosStaticFilesServer.csproj", "{3C7F43BB-FE4C-48BC-BF42-D24E70E8FCB7}"
@@ -36,10 +34,6 @@ Global
                {326BFB1B-5571-47A6-8513-1FFDB32D53B0}.Debug|Any CPU.Build.0 = Debug|Any CPU
                {326BFB1B-5571-47A6-8513-1FFDB32D53B0}.Release|Any CPU.ActiveCfg = Release|Any CPU
                {326BFB1B-5571-47A6-8513-1FFDB32D53B0}.Release|Any CPU.Build.0 = Release|Any CPU
-               {25A82A2A-35C2-4EE0-A0E8-DFDD77978DDA}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
-               {25A82A2A-35C2-4EE0-A0E8-DFDD77978DDA}.Debug|Any CPU.Build.0 = Debug|Any CPU
-               {25A82A2A-35C2-4EE0-A0E8-DFDD77978DDA}.Release|Any CPU.ActiveCfg = Release|Any CPU
-               {25A82A2A-35C2-4EE0-A0E8-DFDD77978DDA}.Release|Any CPU.Build.0 = Release|Any CPU
                {67B1461D-E215-4BA8-A64D-E1836724D5E6}.Debug|Any CPU.ActiveCfg = Debug|Any CPU
                {67B1461D-E215-4BA8-A64D-E1836724D5E6}.Debug|Any CPU.Build.0 = Debug|Any CPU
                {67B1461D-E215-4BA8-A64D-E1836724D5E6}.Release|Any CPU.ActiveCfg = Release|Any CPU
```

Removing this from the solution is required to build, as the code is no longer working.

## 2. Remove UseConsoleLifetime()

```diff
         return Host.CreateDefaultBuilder(args)
             .UseSystemd()
-            .UseConsoleLifetime()
             .ConfigureAppConfiguration((ctx, config) =>
             {
```

This appears in `Program.cs` for all four server programs. Removing UseConsoleLifetime is required for systemd to function correctly.

## 3. Exclude appsettings.json from AuthService build

```diff
--- a/MareSynchronosServer/MareSynchronosAuthService/MareSynchronosAuthService.csproj
+++ b/MareSynchronosServer/MareSynchronosAuthService/MareSynchronosAuthService.csproj
@@ -6,6 +6,18 @@
     <ImplicitUsings>enable</ImplicitUsings>
   </PropertyGroup>
 
+  <ItemGroup>
+    <Content Remove="appsettings.Development.json" />
+    <Content Remove="appsettings.json" />
+  </ItemGroup>
+
+  <ItemGroup>
+    <None Include="appsettings.Development.json" />
+    <None Include="appsettings.json">
+      <CopyToOutputDirectory>Never</CopyToOutputDirectory>
+    </None>
+  </ItemGroup>
+
```

This excludes a dummy appsettings.json file getting copied in to your build output / publish directory, which may potentially clobber your existing configuration!

