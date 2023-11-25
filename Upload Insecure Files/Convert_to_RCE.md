# Agnostic File Upload > RCE vectors

## Linux
### User-writable
```sh
# With an SSH key and access through SSH
~/.ssh/authorized_keys
~/.ssh/authorized_keys2

# With bash files
~/.bash_profile         # Executed for login shells
~/.bash_login           # Executed when a login shell starts
~/.bash_logout          # Executed when a login shell exits
~/.bashrc               # Executed for interactive shells
```

### Root-writable
```sh
# With an SSH key and access through SSH
/root/.ssh/authorized_keys
/root/.ssh/authorized_keys2

# With bash files
/etc/profile            # Executed for all login shells
/etc/bash.bashrc        # Executed for all interactive shells
/etc/bash.bash.login    # Executed when any login shell starts
/etc/bash.bash.logout   # Executed when any login shell exits

# With cron task
/etc/crontab            # Main crontab fragment
/etc/cron.d/*           # Crontab fragment
/etc/cron.hourly/*      # Bash script with shebang
```

## Windows