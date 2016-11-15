# Sudospawner for Jupyterhub or how I learned to hate security


Prepare for a rollercoaster ride. This gets bumpy.

## Install

     sudo pip install jupyterhub
     sudo pip install git+https://github.com/jupyter/sudospawner

## Config

    sudo jupyterhub -y True --generate-config --config=/etc/jupyterhub/jupyterhub_config.py
    
Create a user "rhea" as the system user that will run jupyterhub


## Sudo

We need to enable the user (rhea) to run sudo

Add/replace this into /etc/sudoers
 
    Defaults    secure_path = /sbin:/bin:/usr/sbin:/usr/bin:/opt/anaconda3/bin
    #Defaults    requiretty

In effect, we add the anaconda bin dir to the path that will always be available for sudo commands. This is nessesary.
And we drop the requirement that you must have a tty. This is nessesary. It would be nicer if this could be done from FreeIPA, but....

This could probably be setup for specific users instead of a global setting...

Oh, and rhea must have passwordless sudo. Possible it is enough to give her access to sudospawner, but this works

sudo -U rhea -l

    Matching Defaults entries for rhea on this host:
        !visiblepw, always_set_home, env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE INPUTRC KDEDIR LS_COLORS", env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE", env_keep+="LC_COLLATE
        LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES", env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE", env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
        secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin\:/opt/anaconda3/bin
    
    User rhea may run the following commands on this host:
        (%biusers : ALL) NOPASSWD: /opt/anaconda3/bin/sudospawner
        (ALL : ALL) NOPASSWD: ALL


## DBUS

If you get errors like this in /var/log/messages

    [system] Rejected send message, 2 matched rules; type="method_call", sender=":1.2292" (uid=1031 pid=59107 comm="/opt/anaconda3/bin/python /opt/anaconda3/bin/jupyt") interface="org.freedesktop.login1.Manager" member="CreateSession" error name="(unset)" requested_reply="0" destination="org.freedesktop.login1" (uid=0 pid=823 comm="/usr/lib/systemd/systemd-logind ")

You need to enable the user rhea to handle login sessions 

Create /etc/dbus-1/system.d/jupyterhub.conf
    
    <?xml version="1.0"?> <!--*-nxml-*-->
    <!DOCTYPE busconfig PUBLIC "-//freedesktop//DTD D-BUS Bus Configuration 1.0//EN"
            "http://www.freedesktop.org/standards/dbus/1.0/busconfig.dtd">
    <busconfig>
            <policy user="rhea">
                    <allow send_destination="org.freedesktop.login1"/>
                    <allow receive_sender="org.freedesktop.login1"/>
            </policy>
    </busconfig>
    
In effect, this allows the jupyterhub user (rhea) to handle logins (sessions). Otherwise only root processes could handle login.
 
Restart the dbus deamon

    sudo service dbus restart
    
This disconnects the login daemon.... So restart it

    sudo systemctl restart systemd-logind

    
## PAM

JupyterHub authenticates as the "login" pam service. I feel that it is a lot more clear to make a separate pam service for jupyterhub

    sudo cp /etc/pam.d/login /etc/pam.d/jupyterhub
Then /etc/jupyterhub/jupyterhub_config.py in uncomment this line

    c.PAMAuthenticator.service = 'jupyterhub'

    
## Systemctl 

First, we register it as a systemctl unit service. So create 
 /lib/systemd/system/jupyterhub.service

    [Unit]
    Description=Jupyterhub
    After=network-online.target
    
    [Service]
    User=rhea
    
    #Two ways of specifying the runtime directory. Dunno which works
    RuntimeDirectory=jupyterhub
    WorkingDirectory=/var/run/jupyterhub
    
    
    #This is so the ExecStartPre commands gets run as root
    PermissionsStartOnly=true
    
    #This sets up the logging directory
    ExecStartPre=/usr/bin/mkdir -p /var/log/jupyterhub
    ExecStartPre=/usr/bin/touch /var/log/jupyterhub/jupyterhub.log
    ExecStartPre=/usr/bin/chown rhea /var/log/jupyterhub -R
    
    #Set the path to include anaconda3
    Environment="PATH=/opt/anaconda3/bin/:/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"
    
    
    ExecStart=/opt/anaconda3/bin/jupyterhub --config=/etc/jupyterhub/jupyterhub_config.py --no-ssl --log-file=/var/log/jupyterhub/jupyterhub.log --ip="*" --JupyterHub.spawner_class=sudospawner.SudoSpawner
    
    
    [Install]
    WantedBy=multi-user.target


Load, restart and enable the new service

    sudo systemctl daemon-reload
    sudo systemctl stop jupyterhub
    sudo systemctl start jupyterhub
    sudo systemctl status jupyterhub -l
    sudo systemctl enable jupyterhub


