# How to reproduce

## Steps
- Ensure your SSH key is added to your instance at WPEngine
- Run it with a simple `ansible-playbook --inventory inventory/all ping.yml`

You will see the following error:
```
PLAY [all] *************************************************************************************************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************************************
fatal: [selahomes.ssh.wpengine.net]: UNREACHABLE! => {"changed": false, "msg": "Authentication or permission failure. In some cases, you may have been able to authenticate and did not have permissions on the target directory. Consider changing the remote tmp path in ansible.cfg to a path rooted in \"/tmp\". Failed command was: ( umask 77 && mkdir -p \"` echo ~/.ansible/tmp/ansible-tmp-1573491864.993851-29960961753780 `\" && echo ansible-tmp-1573491864.993851-29960961753780=\"` echo ~/.ansible/tmp/ansible-tmp-1573491864.993851-29960961753780 `\" ), exited with result 1", "unreachable": true}
```

- You can dig into this by using verbose mode:

    ansible-playbook -vvvv --inventory inventory/all ping.yml

In the output, you will see what SSH commands it is trying to execute, for instance:

    <selahomes.ssh.wpengine.net> SSH: EXEC ssh -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o ConnectTimeout=100 -o ControlPath=/Users/jroel/.ansible/cp/f8fc439d22 selahomes.ssh.wpengine.net '/bin/sh -c '"'"'echo ~ && sleep 0'"'"''

- By executing this outside of the Ansible environment, you can more closely see the debug errors that come out of it.
  Remember to add a verbose flag (`-vvvv`) and remove the control path

    ssh -vvvv -C -o ControlMaster=auto -o ControlPersist=60s -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o ConnectTimeout=100  selahomes.ssh.wpengine.net '/bin/sh -c '"'"'echo ~ && sleep 0'"'"''

You can see it is sending the correct command: `debug1: Sending command: /bin/sh -c 'echo ~ && sleep 0'`
But even that simple example does not work (it does not output `/home/wpe-user` as expectedf)

The more complicated example also does not work and you get some better errors from the console:

   ssh -vvv -C -o ControlMaster=auto -o ControlPersist=60s -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o ConnectTimeout=100 selahomes.ssh.wpengine.net '/bin/sh -c '"'"'( umask 77 && mkdir -p "` echo ~/.ansible/tmp/ansible-tmp-1573375538.11-73701803466215 `" && echo ansible-tmp-1573375538.11-73701803466215="` echo ~/.ansible/tmp/ansible-tmp-1573375538.11-73701803466215 `" ) && sleep 0'"'"''

In the output you can see the mismatch:
```
debug1: Sending command: /bin/sh -c '( umask 77 && mkdir -p "` echo ~/.ansible/tmp/ansible-tmp-1573375538.11-73701803466215 `" && echo ansible-tmp-1573375538.11-73701803466215="` echo ~/.ansible/tmp/ansible-tmp-1573375538.11-73701803466215 `" ) && sleep 0'
... some debug lines ...
bash: -c: line 0: syntax error near unexpected token `('
bash: -c: line 0: `/bin/sh -c ( umask 77 && mkdir -p "` echo ~/.ansible/tmp/ansible-tmp-1573375538.11-73701803466215 `" && echo ansible-tmp-1573375538.11-73701803466215="` echo ~/.ansible/tmp/ansible-tmp-1573375538.11-73701803466215 `" ) && sleep 0'

## Another way to see where it is broken
You also see the error when you intentially send a bad command that ends up working:

    ssh -vvvv -C -o ControlMaster=auto -o ControlPersist=60s -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey -o PasswordAuthentication=no -o ConnectTimeout=100  selahomes.ssh.wpengine.net '/bin/sh -c '"'"'whoami echo ~ && sleep 0'"'"''

The `whoami echo` is invalid syntax, but instead of responding with an error, WPEngine's response is `wpe-user`.

That shows that the quotes are removed and the command is eventually received like this:

   /bin/sh -c whoami echo ...

Since there are no quotes, the argument to -c is only `whoami` which gets executed without issues.

## The requested fix

Since being able to use SSH reliable is important for all kinds of automation tools (but specifically Ansible), I would request that WPEngine does not change the output of the request SSH command by stripping quotes, but pass it along as intended.


