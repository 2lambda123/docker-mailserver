---
title: 'Advanced | iOS Mail Push Support'
---

## Introduction

iOS Mail currently does not support the IMAP idle extension. Therefore users can only either check manually or configure intervals for fetching mails in their mail account preferences when using the default configuration.

To support mail push Dovecot needs to advertise the `XAPPLEPUSHSERVICE` IMAP extension as well as sending the actual push notifications to the Apple Push Notification service (APNs) which will forward them to the device.

This can be done with two components:

- A Dovecot plugin (`dovecot-xaps-plugin`) which is triggered whenever a mail is created or moved from/to a mail folder.
- A daemon service (`dovecot-xaps-daemon`) that manages both the device registrations as well as sending notifications to the APNs.

## Prerequisites

- An Apple developer account to create the required Apple Push Notification service certificate.
- Knowledge creating Docker images, using the command-line, and creating shell scripts.

## Limitations

- You need to maintain a custom `docker-mailserver` image.
- Push support is limited to the INBOX folder. Changes to other folders will not be pushed to the device regardless of the configuration settings.
- You currently cannot use the same account UUID on multiple devices. This means that if you use the same backup on multiple devices (e.g. old phone / new phone) only one of them will get the notification. Use different backups or recreate the mail account.

## Privacy concerns

- The service does not send any part of the actual message to Apple.
- The information sent contains the device UUID to notify and the (on-device) account UUID which was generated by the iOS mail application when creating the account.
- Upon receiving the notification, the iOS mail application will connect to the IMAP server given by the provided account UUID and fetch the mail to notify the user.
- Apple therefore does not know the mail address for which the mail was received, only that a specific account on a specific device should be notified that a new mail or that a mail was moved to the INBOX folder.

## Installation

Both components will be built using Docker and included into a custom `docker-mailserver` image. Afterwards the required configuration is added to `docker-data/dms/config`. The registration data is stored in `/var/mail-state/lib-xapsd`.

1. Create a Dockerfile to build a `docker-mailserver` image that includes the [`dovecot-xaps-plugin`](https://github.com/freswa/dovecot-xaps-plugin) as well as the [`dovecot-xaps-daemon`](https://github.com/freswa/dovecot-xaps-daemon). This is required to ensure that the Dovecot plugin is built against the same Dovecot version. The `:edge` tag is used here, but you might want to use a released version instead.

    ```dockerfile
    FROM mailserver/docker-mailserver:edge AS dovecot-plugin-xaps
    WORKDIR /tmp/dovecot-xaps-plugin
    RUN <<EOF
        apt-get update
        apt-get -y --no-install-recommends install git cmake make build-essential dovecot-dev
        git clone --single-branch --depth=1 https://github.com/freswa/dovecot-xaps-plugin.git .
        mkdir build && cd build
        cmake .. -DCMAKE_BUILD_TYPE=Release
        make install
    EOF

    # Use an older Go version as Go >= 1.20 causes this issue: https://github.com/freswa/dovecot-xaps-daemon/issues/24#issuecomment-1483876081
    # Note that the underlying issue are non-standard-compliant Apple http servers which might get fixed at some point
    FROM golang:1.19-alpine AS dovecot-xaps-daemon
    ENV GOPROXY=https://proxy.golang.org,direct
    ENV CGO_ENABLED=0
    WORKDIR /go/dovecot-xaps-daemon
    RUN <<EOF
        apk add --no-cache --virtual build-dependencies git
        git clone --single-branch --depth=1 https://github.com/freswa/dovecot-xaps-daemon .
        go build ./cmd/xapsd
    EOF

    FROM mailserver/docker-mailserver:edge
    COPY --from=dovecot-plugin-xaps /usr/lib/dovecot/modules/*_xaps_* /usr/lib/dovecot/modules/
    COPY --from=dovecot-xaps-daemon /go/dovecot-xaps-daemon/xapsd /usr/bin/xapsd

    # create a non-root user for the daemon process as well as configuration and run state directories
    RUN <<EOF
        adduser --quiet --system --group --disabled-password --home /var/mail-state/lib-xapsd --no-create-home xapsd
        mkdir -p /var/run/xapsd /etc/xapsd
    EOF
    ```

2. Build the new image:
    ```sh
    docker build -t yourname/docker-mailserver .
    ```

3. Modify your `compose.yaml` to use the newly created image:
    ```yaml
        services:
            mailserver:
              image: yourname/docker-mailserver:latest
    ```

4. Recreate the container:
    ```sh
    docker compose down
    docker compose up -d
    ```

5. Create a hash of your Apple developer account password using the provided `xapsd -pass` command:
    ```sh
    docker exec -it mailserver xapsd -pass
    ```

6. Add configuration for both components:
    - Create a folder named `xaps` in `docker-data/dms/config`.

    - Create a file named `xapsd.yaml` in `docker-data/dms/config/xaps`. 
        - Replace `appleId` and `appleIdHashedPassword` with your actual credentials. For reference see also [here](https://github.com/freswa/dovecot-xaps-daemon/blob/master/configs/xapsd/xapsd.yaml).
        - The service will use the provided username/hash combination to automatically request a new certificate from Apple as well as renewing an older certificate if needed.

        ```yaml title="xapsd.yaml"

        # set the loglevel to either
        # trace, debug, error, fatal, info, panic or warn
        # Default: info
        loglevel: info

        # xapsd creates a json file to store the registration persistent on disk.
        # This sets the location of the file.
        databaseFile: /var/mail-state/lib-xapsd/database.json

        # xapsd listens on a socket for http/https requests from the dovecot plugin.
        # This sets the address and port number of the listen socket.
        listenAddr: '127.0.0.1'
        port: 11619

        # xapsd is able to listen on a HTTPS Socket to allow HTTP/2 to be used
        # SSL is enabled implicitly when certfile and keyfile exist
        # !!! only use HTTPS for connection pooling with a proxy e.g. nginx or HaProxy
        # !!! direct usage with the plugin is discouraged and unsupported
        tlsCertfile:
        tlsKeyfile:
        tlsListenAddr:
        tlsPort: 11620

        # Notifications that are not initiated by new messages are not sent immediately for two reasons:
        # 1. When you move/copy/delete messages you most likely move/copy/delete more messages within a short period of time.
        # 2. You don't need your mailboxes to synchronize immediately since they are automatically synchronized when opening
        #    the app
        # If a new message comes and the move/copy/delete notification is still on hold it will be sent with the notification
        # for the new message.
        # This sets the interval to check for delayed messages.
        checkInterval: 20

        # Set the time how long notifications for not-new messages should be delayed until they are sent.
        # Whenever checkInterval runs, it checks if "delay" <= "waiting time" and sends the notification if the expression is
        # true.
        delay: 30

        # To retrieve certificates from Apple, we need to login with a valid Apple ID
        # The accounts email must be given in cleartext, but the password has to
        # be hashed before sending it. To not leak working credentials on running servers,
        # we do not accept the cleartext password here.
        appleId: foo@example.com

        # use `xaps -pass` to calculate the hash of the apple id password
        appleIdHashedPassword: bar
        ```

    - Create a file named `95-xaps.conf` in `docker-data/dms/config/xaps`. For reference see also [here](https://github.com/freswa/dovecot-xaps-plugin/blob/master/xaps.conf).
        ```txt title="95-xaps.conf"

        protocol imap {
          mail_plugins = $mail_plugins notify push_notification xaps_push_notification xaps_imap
        }

        protocol lda {
          mail_plugins = $mail_plugins notify push_notification xaps_push_notification
        }

        protocol lmtp {
          mail_plugins = $mail_plugins notify push_notification xaps_push_notification
        }

        plugin {
            # xaps_config contains xaps specific configuration parameters
            # url:              protocol, hostname and port under which xapsd listens
            # user_lookup: Use if you want to determine the username used for PNs from environment variables provided by
            #                   login mechanism. Value is variable name to look up.
            # max_retries:      maximum num of retries the http client connects to the xaps daemon
            # timeout_msecs     http timeout of the http connection
            xaps_config = url=http://127.0.0.1:11619 user_lookup=theattribute max_retries=6 timeout_msecs=5000
            push_notification_driver = xaps
        }
        ```

    - Create a supervisord file named `xapsd.conf` in `docker-data/dms/config/xaps` with the following content:
        ```txt title="xapsd.conf"

        [program:xapsd]
        startsecs=0
        autostart=false
        autorestart=true
        stdout_logfile=/var/log/supervisor/%(program_name)s.log
        stderr_logfile=/var/log/supervisor/%(program_name)s.log
        user=xapsd
        command=/usr/bin/xapsd
        pidfile=/var/run/xapsd/xapsd.pid
        ```

    - Create or update your `user-patches.sh` in `docker-data/dms/config` to move the files to their final location as well as starting the daemon service:
        ```sh title="user-patches.sh"
        #!/bin/bash

        # Copy the configs to internal locations:
        cp /tmp/docker-mailserver/xaps/95-xaps.conf /etc/dovecot/conf.d/95-xaps.conf
        cp /tmp/docker-mailserver/xaps/xapsd.yaml /etc/xapsd/xapsd.yaml
        cp /tmp/docker-mailserver/xaps/xapsd.conf /etc/supervisor/conf.d/xapsd.conf

        # Setup data persistence and ensure ownership is always for xapsd:
        mkdir -p /var/mail-state/lib-xapsd
        chown -R xapsd:xapsd /var/mail-state/lib-xapsd

        # Start the xaps daemon:
        supervisorctl update
        supervisorctl start xapsd
        ```

7. Recreate the container again to apply the new configuration:
    ```sh
    docker compose down
    docker compose up -d
    ```

8. Recreate your mail account on your iOS device and check the logs in `/var/log/supervisor/dovecot.log` and `/var/log/supervisor/xapsd.log` for any errors.


## Other configuration options

Both device registration and notifications send a username to the daemon to lookup the device. While the registration and other IMAP operations in Dovecot will send the Dovecot username, LMTP will send the provided authentication username.

The format of that username is specified by the `auth_username_format` Dovecot setting. If you are not using mail addresses as Dovecot usernames - e.g. when using LDAP - you can either change the `auth_username_format` or add the mail address as property to the user account and use the lookup feature (see below).

```sh title="user-patches.sh"
sed -i -r "s|^#?(auth_username_format =).*|\1 %Ln|" /etc/dovecot/conf.d/10-auth.conf
```

You can also use notifications for Dovecot alias mailboxes. Depending on your server configuration, this might require to add the original Dovecot username as [Dovecot attribute](https://doc.dovecot.org/configuration_manual/authentication/user_database_extra_fields/) to the login user as well as changing the `user_lookup=theattribute` in `95-xaps.conf` to perform the lookup of that attribute.
