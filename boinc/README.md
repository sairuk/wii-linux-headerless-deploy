# boinc

This will setup the boinc-client on the wiis, may need to massage it a bit from
my setup.

whatever project you choose will need to support PPC workloads, this limits the 
options quite a bit.

update your inventory in ``` inventory ``` , it wont match mine

edit ```vars/boinc_vars.yml``` to match your setup

run the boinc.yaml playbook, e.g.

``` ansible all -b boinc.yml ```

We need to fake the memory info for the wii since the project I am subbed with 
requires a minimum of 95mb to accept workloads, we do that by mounting a fake
file over /proc/meminfo ... dodgy yes but also effective.


You can then run adhoc commands against the wii's to query the tasks (or put it 
in a playbook, whatever gets you out of bed in the morning)

``` ansible all -a 'boinccmd --get_tasks' ```

If you see ``` active_task_state: EXECUTING ``` give yourself +1 on you hackery
skill set


All things considered you should also be able to review my subs to the wuprop 
project [here] (https://wuprop.boinc-af.org/hosts_user.php?userid=134073)

sairuk
