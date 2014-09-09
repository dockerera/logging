logging
=======



IMVHO I think a syslog plugin for docker is completely pointless. Syslog daemons that run on practically every distribution already support everything that has been discussed here. This comment outlines precisely why I feel this way. This is not meant as a flame, but as an alternative discussion that relieves some pressure off the docker devs and puts more responsibility on the Dockerfile maintainer where in this case I feel (again, IMVHO) it belongs.

I am not saying that the logging doesn't need some work; the piece about handling/rotating the stderr/stdout of the container itself is incredibly useful b/c for long-running containers pushing a lot of logs to those pipes results in the issues previously described regarding disk usage. This will at some point need to be solved, though, to cover the bevy of trusted builds that currently send everything to stderr/stdout.
configuring syslog output within the container

I find that the following options work beautifully (these should obviously be expanded):

    Most apps themselves provide syslog output via configuration. If one doesn't, it should (and probably can be setup but just isn't documented very well). This is especially true for java apps via slf4j, logback, log4j, etc... etc.. Dockerfiles should modify/ADD correct syslog daemon configuration endpoints. My example is for elasticsearch's logging config (this is for @LK4D4) usually found in config/logging.yaml. The conversionPattern could be mangled by a startup script via sed, etc.. to throw-in the container id, hostname, whatever you want (instead of elasticsearch:, or perhaps nothing if you are fine with just the hostname showing up in the syslog). Here is the relevant snippet (I didn't include the default console appender and level in this example):

rootLogger: INFO, syslog
appender:
  syslog:
    type: syslog
    header: true
    syslogHost: <THE_HOST_SYSLOG_DAEMON>
    Facility: USER
    layout:
      type: pattern
      conversionPattern: "elasticsearch: [%p] %t/%c{1} - %m"

    A wrapper startup script that exec's out everything to logger. For non-daemonized processes running in a wrapper, this just magically works and handles stderr/stdout appropriately (written as a boilerplate that can be modified to run in a sourced file easily)

#!/bin/bash

if host syslog > /dev/null 2>&1; then HOST_SYSLOG_DAEMON=$(host syslog | head -n 1 | cut -f 4); fi

_enable_syslog=${SYSLOG:-true}
_host_syslog_daemon="${HOST_SYSLOG_DAEMON:-172.17.42.1}" # perhaps loaded by --env/-e; ${VAR:-DEFAULT} notation sets default if ENV variable does not exist
_unique_proc_name="randywallace/test_syslog_image"
_facility='local0' # or local1 thru local7, cron, user, etc...
_syslog_and_stdout_stderr=${TEE_OUTPUT:-false} # true/false; also could be a --env

# docker run -e TEE_OUTPUT=true -e HOST_SYSLOG_DAEMON=1.2.3.4 -d my_image
# or
# docker run --link my-rsyslog-container:syslog -d my_image
# or disabled completely
# docker run -e SYSLOG=false -d my_image

__logger() {
  local LEVEL=${1:-info}
  sed -u -r -e 's/\\n/ /g' -e 's/\s\-{3,}/;/g' -e 's/\-{3,}\s//g' |\
  /usr/bin/logger -p ${_facility}.${LEVEL}  -t "${_unique_proc_name}[$$]" -n "${_host_syslog_daemon}"
}

run_logger() {
  if $_enable_syslog; then
    if $_syslog_and_stdout_stderr; then
      tee -a >(__logger $1)
    else
      __logger $1
    fi
  else
    if [ "$1" = "err" ]; then
      cat >&2
    else
      cat
    fi
  fi
}

# Catch all STDOUT and STDERR traffic
exec > >(run_logger info) 2> >(run_logger err)

log() { echo "INFO: $*" | run_logger ; }
error() { echo "ERROR: $*" | run_logger err ; }
critical() { echo "EMERGENCY: $*" | run_logger emerg; exit 1 ; }
alert() { echo "ALERT: $*" | run_logger alert ; }
notice() { echo "NOTICE: $*" | run_logger notice ; }
debug() { echo "DEBUG: $*" | run_logger debug ; }
warning() { echo "WARN: $*" | run_logger warn ; }

log "info"
error "error"
alert "alert"
notice "notice"
debug "debug"
warning "warning"

echo "STDOUT output"
echo "STDERR output" >&2

critical "critical... exiting"

identifying the host to receive syslog traffic

    use an ENV setting in the dockerfile (see wrapper example above) to indicate your preferred default syslog host. Or, for public Dockerfiles, use a default config (SYSLOG=false in the wrapper above) that is caught at startup to disable syslog output.

    use a docker container with a volume on /var/log to the host (perhaps on /var/log/docker/syslog/ at the host) and a syslog daemon (I use rsyslog personally). Then EXPOSE the syslog port (514) and link that container to your other containers and specify that link alias in your wrapper (no need to specify the dynamic IP b/c it shows up in /etc/hosts, an example is given in the wrapper that uses 'syslog' for the link alias).

    Use the actual host daemon, if there is one (boot2docker does not have a syslog daemon, so I use a container and volume). This defaults to 172.17.42.1 unless docker is configured differently, but I don't ever need to change that so I set this IP statically. It would be nice if the docker0 Bridge Gateway IP was configured in /etc/hosts on the containers so that I could specify that in cases in which I may need to change the bridge subnet or something. It may already be there, food for thought.

Profit

    The hostname of the container shows up in the syslog in all cases. Why not set this when you run the container to something useful? If you're forwarding logs from syslog to logstash/splunk/etc... The IP of the forwarding syslog server will show up, so you can always identify where container X came from.

    The syslog daemon does not have to exist on the same host as the container. Why fight with tailing /var/log/syslog on 10 docker hosts if you could do it on one?

    You can use syslog daemon configs to do whatever you want with that stuff getting thrown at it to include
        Redirect container logs to a different file in /var/log than the host config
        send critical/alert/notify messages to all the consoles (http://superuser.com/questions/515337/rsyslogd-how-to-send-messages-to-user-pseudo-console)... wouldn't it be nice to get a console message when a container has finished it's 'bootup'?
        send critical/warning messages to your monitoring framework from the host/syslog container without having to screw with the internals of the container config: (http://www.rsyslog.com/doc/omprog.html)

Conclusion

Solving the problem of logging is not a new one, and I seriously doubt that docker could create enough plugins, command line options, etc... to satisfy everybody. This is why rsyslog, syslog-ng, syslogd, papertrail, logstash, graylog2, splunk, fluentd, etc... exist. We've already seen this battle start here, and I don't want to be around when the smoke clears. I hope what I've said here, though, may help some of you to come up with your own solutions that could be working today!

And, if you have problems with the container's logs getting too full (those that are generated from stderr/stdout), don't send them there at all and use my example wrapper above to get rid of that problem completely!

