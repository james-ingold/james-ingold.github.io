---
title: Migrating from Sequel Pro to Sequel Ace
description: Moving on from Sequel Pro
author: James Ingold
published: true
---

Sequel Pro is a MySQL/MariaDB management GUI for macOS and it has been around for a long time. Sequel Pro's first release was in 2008 and at the time of this writing, the last one was in 2016. I've been using Sequel Pro for the past seven years but with the update to Big Sur, it's time to move on. Sequel Pro has become slow to use but I've continued on using it as a habit. A co-worker mentioned that the project forked into one called Sequel Ace. It's still [open source]("https://github.com/Sequel-Ace/Sequel-Ace") and actively maintained. Here's how to install Sequel Ace and pull over your favorite database connections from Sequel Pro with the least amount of headache.

#### Install Sequel Ace

`brew install --cask sequel-ace`

and if you have a Brewfile, add the cask "sequel-ace"

#### Copy Preferences and Favorites

```
cp ~/Library/Preferences/com.sequelpro.SequelPro.plist ~/Library/Containers/com.sequel-ace.sequel-ace/Data/Library/Preferences/com.sequel-ace.sequel-ace.plist
```

```
cp ~/Library/Application\ Support/Sequel\ Pro/Data/Favorites.plist ~/Library/Containers/com.sequel-ace.sequel-ace/Data/Library/Application\ Support/Sequel\ Ace/Data/Favorites.plist
```

#### Copy Passwords (optional)

The database connections that we copied over will not have the passwords because those are stored in the Keychain Access app. You can re-enter those passwords and save them as you connect to the different databases or you can open KeyChain Access and copy them over ahead of time. Open up KeyChain Access and search for "Sequel Pro". From there, you can click on the different connections and then check "Show Password" to view the plain text password. Next, copy that password over to Sequel Ace.

If you have many favorite connections, Sequel Ace offers the following bash script:

```bash
#!/bin/bash

results=$(security dump-keychain -r | grep "Sequel Pro" | sed -E -e 's/^.*"([^"]+)"$/\1/' | sort | uniq)
IFS=$'\n' read -r -d '' -a items < <(printf '%s\0' "$results")

for key in "${items[@]}"
    do
        newkey=$(echo "$key" | sed -E -e 's/Sequel Pro/Sequel Ace/')
        echo "Migrating: $key --> $newkey"
        account=$(sudo security find-generic-password -l "$key" | grep "acct\"<blob>" | sed -E -e 's/^.*"acct"<blob>="(.+)"$/\1/')
        pwd=$(sudo security find-generic-password -l "$key" -w)
        security add-generic-password -a "$account" -s "$newkey" -w "$pwd" -T "/Applications/Sequel Ace.app" -U
        echo "Done: $key --> $newkey"
done
```

#### Uninstall Sequel Pro

`brew uninstall --cask sequel-pro`

#### Wrapping Up

Now we've got our Sequel Pro connections moved over to Sequel Ace, an actively maintained database GUI ready for the next decade of coding on a Mac!

##### References:

[Migrating From Sequel Pro](https://sequel-ace.com/get-started/migrating-from-sequel-pro.html){:target="\_blank"}
