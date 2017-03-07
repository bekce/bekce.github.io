---
layout:     post
title:      "How to create a linux service"
date:       2015-08-11 22:32:00
categories: sysadmin
---

Usually you need to create a linux service from a command. Here I present a simple and efficient init.d template script and recall commands to enable/disable your service on system startup. You need to supply four basic parameters:

*   Service name
*   Full path to executable (with arguments)
*   Working directory
*   User name for the spawned process

    sudo nano /etc/init.d/<service_name>

```shell
#!/bin/sh
### BEGIN INIT INFO
# Provides:
# Required-Start:    $remote_fs $syslog
# Required-Stop:     $remote_fs $syslog
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: Start daemon at boot time
# Description:       Enable service provided by daemon.
### END INIT INFO

# Fill this part
name="<service_name>"
cmd="<executable_with_parameters>"
dir="<working_directory>"
user="<process_owner>"

pid_file="/var/run/$name.pid"
stdout_log="/var/log/$name.log"
stderr_log="/var/log/$name.err"

get_pid() {
    cat "$pid_file"
}

is_running() {
    [ -f "$pid_file" ] && ps `get_pid` > /dev/null 2>&1
}

case "$1" in
    start)
    if is_running; then
        echo "Already started"
    else
        echo "Starting $name"
        cd "$dir"
        if [ -z "$user" ]; then
            sudo $cmd >> "$stdout_log" 2>> "$stderr_log" &
        else
            sudo -u "$user" $cmd >> "$stdout_log" 2>> "$stderr_log" &
        fi
        echo $! > "$pid_file"
        if ! is_running; then
            echo "Unable to start, see $stdout_log and $stderr_log"
            exit 1
        fi
    fi
    ;;
    stop)
    if is_running; then
        echo -n "Stopping $name.."
        kill `get_pid`
        for i in {1..10}
        do
            if ! is_running; then
                break
            fi

            echo -n "."
            sleep 1
        done
        echo

        if is_running; then
            echo "Not stopped; may still be shutting down or shutdown may have failed"
            exit 1
        else
            echo "Stopped"
            if [ -f "$pid_file" ]; then
                rm "$pid_file"
            fi
        fi
    else
        echo "Not running"
    fi
    ;;
    restart)
    $0 stop
    if is_running; then
        echo "Unable to stop, will not attempt to start"
        exit 1
    fi
    $0 start
    ;;
    status)
    if is_running; then
        echo "Running"
    else
        echo "Stopped"
        exit 1
    fi
    ;;
    *)
    echo "Usage: $0 {start|stop|restart|status}"
    exit 1
    ;;
esac

exit 0
```
Make it executable:

    sudo chmod +x /etc/init.d/<service_name>

Start service:

    sudo service <service_name> start

Logs are here (they don't rotate):

    /var/log/<service_name>.log
    /var/log/<service_name>.err

Register your service to run at system startup:

    # on ubuntu:
    sudo update-rc.d <service_name> defaults
    # on redhat,centos:
    sudo /sbin/chkconfig --add <service_name>
    sudo /sbin/chkconfig <service_name> on

Remove (for future reference): use for removing run level links of your service

    # on ubuntu:
    sudo update-rc.d -f <service_name> remove
    # on redhat,centos:
    sudo /sbin/chkconfig <service_name> off

It is customary to configure [Monit](https://mmonit.com/monit/documentation/monit.html) to keep track of your service process after this step. ([Link](https://raw.githubusercontent.com/fhd/init-script-template/aa41261486c3eb521238fd39e41bf49a74ff629b/template) to original script, I have modified it and fixed a bug.)
