Noven INI Update for Exponential
================================

Noven INI Update switches an Exponential installation's configuration from one environment to
another (local, dev, preprod, prod ...). You describe once, in one XML file, which INI settings,
which `config.php` constants and which cluster settings differ per environment; the extension
then writes the values of the chosen environment into the files, from the **INI Config** tab of
the administration or from the command line, with a diff beforehand and an optional backup.

It is for teams that move one code base between several servers and are tired of editing
`site.ini.append.php`, `file.ini.append.php` and `config.php` by hand after each copy.

Contents
--------

- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [The XML file](#the-xml-file)
- [Usage in the administration](#usage-in-the-administration)
- [Usage on the command line](#usage-on-the-command-line)
- [Permissions](#permissions)
- [Upgrading](#upgrading)
- [Known limits](#known-limits)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)
- [Credits and history](#credits-and-history)

Features
--------

- **INI settings per environment**: strings, numbers, true/false, enabled/disabled and arrays
  (simple and associative), in any INI file, written through the kernel's INI API.
- **config.php**: constants (booleans too) and blocks of custom code per environment.
- **Cluster mode**: the file handler and database settings of `file.ini` and the matching
  `index_cluster.php`, per environment (eZDBFileHandler and eZDFSFileHandler, MySQL and mysqli
  backends).
- **A diff** between the current values and those to be applied (command line).
- **Backups** of the files before they are changed.
- **The current environment** is remembered (in the `ezsite_data` table) and shown.
- **Environment limitation** on the policy, so a role can be allowed only some environments
  (for example developers: dev and preprod only).
- **The INI Config tab** in the administration. In the admin4 design the tab shows the sliders
  icon before its name.
- **A command line script** for deployments.
- English and French texts.

Requirements
------------

- Exponential 6.x, with ezjscore active (it is by default)
- PHP 8.0 or later, with SimpleXML
- MySQL or MariaDB (the current environment is stored with MySQL syntax)
- Write access for the web server user (or the user running the script) to the INI files, to
  `config.php` and to `index_cluster.php` the XML file names

Installation
------------

### With Composer

From the root of the Exponential installation:

```bash
composer require se7enxweb/noveniniupdate
```

The Exponential legacy installer places the package in `extension/noveniniupdate`.

### By hand

1. Download the latest release from https://github.com/se7enxweb/noveniniupdate/releases (or
   `git clone https://github.com/se7enxweb/noveniniupdate.git`).
2. Copy the folder to `extension/noveniniupdate`. The folder name must be `noveniniupdate`.

### Activate it

Add the extension to `settings/override/site.ini.append.php`:

```ini
[ExtensionSettings]
ActiveExtensions[]=noveniniupdate
```

Then regenerate the autoloads and clear the caches, from the installation root:

```bash
php bin/php/ezpgenerateautoloads.php -e
php bin/php/ezcache.php --clear-all
```

Configuration
-------------

`extension/noveniniupdate/settings/noveniniupdate.ini` holds the defaults; do not edit it. Put
your values in `settings/override/noveniniupdate.ini.append.php`:

```ini
<?php /* #?ini charset="utf-8"?

[XmlSettings]
# The XML file that describes the environments, relative to the installation root.
XmlContent=settings/environments.xml

[BackupSettings]
# Where backups go, without a trailing slash; may be outside the installation.
BackupDir=var/noveniniupdate_backup
# enabled: each backup gets its own folder named after the date and time.
DateTimeBackupDir=enabled
# The date() pattern of those folders.
DateTimeBackupDirPattern=Ymd-His

*/ ?>
```

`[XmlSettings] XmlType[]` lists the value types and their labels; the defaults need no change.
The extension's own `noveniniupdate.ini.append.php` points `XmlContent` at the sample file, so
always set your own.

Some INI settings may be read only for the INI API; check `site.ini`
`[eZINISettings] ReadonlySettingList[]` (by default `template.ini/PHP/PHPOperatorList`,
`image.ini/ImageMagick/ExecutablePath` and `image.ini/ImageMagick/Executable`).

The extension also adds to `menu.ini` (the INI Config tab), `module.ini` (the module
`noveniniupdate`), `design.ini`, `site.ini` (translations) and `i18n.ini`.

The XML file
------------

Start from `extension/noveniniupdate/source/sample.xml`. Its structure:

```xml
<?xml version="1.0" encoding="utf-8" standalone="yes"?>
<IniConfig>
    <envs>
        <env name="dev" comment="Development"/>
        <env name="prod" comment="Production"/>
    </envs>

    <files>
        <file path="settings/override/site.ini.append.php" comment="Site settings">
            <line env="dev"  block="SiteSettings" type="string" name="SiteURL" value="dev.example.com" comment=""/>
            <line env="prod" block="SiteSettings" type="string" name="SiteURL" value="www.example.com" comment=""/>

            <line env="dev"  block="DebugSettings" type="enable/disable" name="DebugOutput" value="enabled" comment=""/>
            <line env="prod" block="DebugSettings" type="enable/disable" name="DebugOutput" value="disabled" comment=""/>

            <line env="dev" block="MailSettings" type="array" name="AdminEmailList" comment="Simple array">
<![CDATA[
=dev-team@example.com
]]>
            </line>
        </file>
    </files>

    <!-- Optional: cluster settings, written to the INI file and to index_cluster.php -->
    <ClusterMode ini="settings/override/file.ini.append.php" php="index_cluster.php">
        <cluster env="prod">
            <FileHandler>eZDFSFileHandler</FileHandler>
            <!-- database host, port, name, user, password, mount point ...: see sample.xml -->
        </cluster>
    </ClusterMode>

    <!-- Optional: config.php -->
    <ConfigPHPFile>
        <config env="dev">
            <!-- For booleans use 0/1 and isBoolean="true" -->
            <constant name="EZP_INI_FILEMTIME_CHECK" value="1" isBoolean="true"/>
        </config>
        <config env="prod">
            <constant name="EZP_INI_FILEMTIME_CHECK" value="0" isBoolean="true"/>
        </config>
    </ConfigPHPFile>
</IniConfig>
```

- Every `<line>` names its environment, the INI block, the type (`string`, `numeric`,
  `true/false`, `enable/disable` or `array`), the setting and its value.
- Arrays are written in a CDATA block, one `=value` (or `key=value` for an associative array)
  per line.
- Leave `<ClusterMode>` or `<ConfigPHPFile>` out if you do not need them.
- `sample.xml` shows every case, including both cluster handlers and custom code in config.php.

Usage in the administration
---------------------------

1. Open the **INI Config** tab (`/noveniniupdate/view`). It shows the current environment.
2. Select an environment and press **Select**: the page lists every setting the XML file gives
   it, file by file, with the cluster and config.php settings (the command line's --diff shows
   what would change).
3. Check **Do a backup before switching** if you want a copy of the files first.
4. Press **Update environment with current XML content** and confirm. The files are written and
   the environment is remembered as the current one.
5. Clear the INI cache afterwards (`php bin/php/ezcache.php --clear-tag=ini`) so every process
   reads the new values.

Usage on the command line
-------------------------

From the installation root:

```bash
# The environments of the XML file
php extension/noveniniupdate/bin/php/noviniupdate.php --list-envs

# The settings of one environment
php extension/noveniniupdate/bin/php/noviniupdate.php --env=prod --list-params

# What would change
php extension/noveniniupdate/bin/php/noviniupdate.php --env=prod --diff

# Switch, with a backup first
php extension/noveniniupdate/bin/php/noviniupdate.php --env=prod --backup

# Then let every process read the new values
php bin/php/ezcache.php --clear-tag=ini
```

A deployment script can run the switch right after the code is copied to a server.

Permissions
-----------

The tab and the view need the policy **noveniniupdate/configupdate**. Its limitation
**Environment** (`NovenINIUpdate_Environment`) restricts the environments a role may apply;
without a limitation every environment is allowed. Changing configuration files is as powerful as
editing them on the server: give this policy only to people you trust with that.

Upgrading
---------

1. Replace the extension's folder with the new release (`composer update se7enxweb/noveniniupdate`,
   or copy the new files over the old ones).
2. Regenerate the autoloads and clear the caches (see [Activate it](#activate-it)).
3. Read [doc/CHANGELOG](doc/CHANGELOG).

Known limits
------------

- The values of the XML file are written as they are; keep secrets (database passwords) in the
  XML file only if the file is protected like the INI files themselves.
- The current environment is stored with MySQL syntax; other database engines are not supported.
- `config.php` and `index_cluster.php` are rewritten from the XML values, so hand-made changes to
  the parts the XML file manages are replaced.
- The admin page is made for the admin designs.

Troubleshooting
---------------

- **The tab is missing.** Check `ActiveExtensions[]`, clear the caches, and give your role
  `noveniniupdate/configupdate`.
- **The tab has no icon.** Icons are drawn by the admin4 design; older admin designs show the
  name only.
- **"Given environment ... is not supported/declared in XML config file".** The name must match an
  `<env name="...">` of the XML file.
- **A setting does not change.** It may be in `ReadonlySettingList[]`, or the file is not writable
  for the user that runs the update.
- **The site still uses the old values.** Clear the INI cache (`--clear-tag=ini`).

Bug reports: https://github.com/se7enxweb/noveniniupdate/issues

Contributing
------------

Pull requests are welcome at https://github.com/se7enxweb/noveniniupdate. Translations go in
`translations/<locale>/translation.ts`. The original documentation is in
`doc/Readme NovenINIUpdate.pdf`; `ant/build.xml` is the original release build.

License
-------

Noven INI Update is licensed under the GNU General Public License v2.0; see [LICENSE](LICENSE).

Credits and history
-------------------

Jean-Luc Nguyen and Jérôme Vieilledent wrote Noven INI Update at Noven between 2009 and 2012 for
eZ Publish, the content management system that preceded Exponential, and published it on
projects.ez.no and GitHub (lolautruche/noveniniupdate). Yannick Olympio and others contributed
fixes. 7x maintains it for Exponential.

- Copyright (C) 2009 - 2012 Jean-Luc Nguyen, Jérôme Vieilledent - Noven
- Copyright (C) 1998 - 2026 7x
