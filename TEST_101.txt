---

Ansible 2.8.6
Tower 2.6.3
Tower CLI 3.3.9
Python 3.6
Python 2.7
CentOS 7

pip3.6 install configparser
pip3.6 install Pygithub
pip3.6 install ansible-tower-cli

wget https://github.com/ArctiqTeam/tower-webhook/archive/master.zip
yum install unzip -y
unzip master.zip
cd tower-webhook-master
vim hosts

**hosts**

[webhook]
webhookserver ansible_host=root@127.0.0.1 ansible_ssh_port=22 ansible_password={{ANSIBLE-TOWER-ADMIN-PASSWORD}}

***

vim vars.yml

|| Note : In your name, do not include spaces, this is the webhook /hooks/NAMEHERE, and %20's just don't do well, so no spaces here
**vars**

---
projects:
  - id: 10
    name: GitHub

*******


ansible-playbook install.yml

vim /etc/webhook/webhook.conf

|| Note :  You pick your password for webhook, make sure all app servers have the same password.  this is the password that you will give GitHub to be allowed to talk to this WebHook!

**webhook.conf**

[
  {
    "id": "GitHub",
    "execute-command": "/usr/local/bin/ansible-tower-ci.py",
    "pass-environment-to-command":
    [
      {
        "source": "payload",
        "name": "action",
        "envname": "GHE_ACTION"
      },
      {
        "source": "payload",
        "name": "pull_request.statuses_url",
        "envname": "GHE_STATUSES_URL"
      },
      {
        "source": "payload",
        "name": "pull_request.url",
        "envname": "GHE_PULL_URL"
      },
      {
        "source": "payload",
        "name": "pull_request.head.repo.ssh_url",
        "envname": "GHE_SSH_URL"
      },
      {
        "source": "payload",
        "name": "pull_request.head.repo.default_branch",
        "envname": "GHE_BRANCH"
      },
      {
        "source": "payload",
        "name": "pull_request.head.sha",
        "envname": "GHE_SHA"
      },
      {
        "source": "payload",
        "name": "number",
        "envname": "GHE_ISSUE_ID"
      }
    ],
    "trigger-rule":
    {
      "match":
      {
        "type": "payload-hash-sha1",
        "secret": "{{ PASSWORD_FOR_WEBHOOK }}",
        "parameter":
        {
          "source": "header",
          "name": "X-Hub-Signature"
        }
      }
    }
  }
]


***********************

systemctl restart webhook 


|| Note:  Make sure your webhook is running, try to telnet to the ip you plan on using ' telnet 127.0.0.1 9000 '
|| Note:  lsof -i:9000 

**vim /usr/local/bin/ansible-tower-ci.py**

#!/usr/bin/env python3.6
import os
import time
import configparser
from github import Github


###########################################################################
# Ansible Tower - GitHub Pipeline
###########################################################################

# Notes
##

def value_grabber(group, item):
    try:
        config = configparser.ConfigParser()
        config.read(PASSWORD_FILE)
        item_to_ret = config.get(group, item)
        return item_to_ret
    except Exception as e:
        print(e)
        return "err"


# Global
try:
    PASSWORD_FILE = (r"/etc/python/.netpass.ini")
    GHE_TOKEN = value_grabber('GitHub', 'token')
    GHE_REPO = value_grabber('GitHub', 'repo')
    GHE_URL = value_grabber('GitHub', 'url')
    TOWER_ADMIN = value_grabber('Tower', 'admin')
    TOWER_PASSWORD = value_grabber('Tower', 'password')
    TOWER_HOST = value_grabber('Tower', 'host')
    TOWER_URL = value_grabber('Tower', 'url')
    TOWER_PROJECT_ID = value_grabber('Tower', 'project_id')
    TOWER_JOB_ID = ""
    TOWER_PROJECT_LOCATION = value_grabber('Tower', 'project_location')
except Exception as e:
    print(e)
    exit()

# Items set by WebHook
try:
    GHE_ACTION = os.environ['GHE_ACTION']
    GHE_STATUSES_URL = os.environ['GHE_STATUSES_URL']
    #GHE_PULL_URL = os.environ['GHE_PULL_URL']
    GHE_SSH_URL = os.environ['GHE_SSH_URL']
    GHE_BRANCH = os.environ['GHE_BRANCH']
    GHE_SHA = os.environ['GHE_SHA']
    GHE_ISSUE_ID = os.environ['GHE_ISSUE_ID']
    #Make sure this was an open pull reuqest vs a close etc.etc
    found_it = False
    for search in ["open","synchronize"]:
        if search in GHE_ACTION:
            #If this is not an open, we don't care.  close
            found_it = True
    if found_it == False:
       print ("Skip Check...")
       exit()


except Exception as e:
    print(e)
    exit()

# Definitions
def os_command(command):
    try:
        os.popen(command)
        os.wait()
        return "pass"
    except Exception as e:
        print(e)
        return "err"


def os_command_return_lines(command):
    # Really this is only used to get the project ID during a tower-cli command..but looks cool?
    try:
        item_to_ret = os.popen(command).readlines()
        item_to_ret = item_to_ret[4].replace("N/A", "").strip()  # Get the Project ID to give better links in github
        os.wait()
        return item_to_ret
    except Exception as e:
        print(e)
        return "err"


def os_command_check_for_error(command, version=""):
    # So this is just for ansible, I can't name def's to save my life.  Execute the ansible playbook, if you see an error dip out and report it
    # TODO - Support multiple versions of ansible to test, not sure how we could pass this from a github payload just yet...future self?
    try:
        output = os.popen(command).readlines()
        success = True
        text_count = 0

        #Because we can't see the output include the word ERROR in a blank .yml  file (Bug#1?) count how many strings were returned.  a running playbook will have more lines
        #than a non-working/running playbook
        for i in output:
            text_count = text_count + 1
        if text_count <= 0:
           os.wait()
           return "err"

        line_count = 0
        #possible we can make everything lowercase.  This gives you the ability to add whatever 'failure' words you want to the test.
        for search in ["ERROR!","error","ERROR!"]:
           line_count = line_count + 1
           if search in output:
               success = False
               os.wait()
               return "err"
        os.wait()
        if success:
           return "pass"
        else:
           #Really I don't know how you got here, but just in case
           return "err"
    except Exception as e:
        print(e)
        return "err"





def githubcommand(ghe_action, ghe_state="", twr_url=TOWER_URL, ghe_description="", ):
    try:
        g = Github(base_url=GHE_URL, login_or_token=GHE_TOKEN)
        repo = g.get_repo(GHE_REPO)
        if ghe_action == 'status_update':
            #print(str(repo))
            repo.get_commit(sha=GHE_SHA).create_status(
                state=ghe_state,  # error,pending,failure,success 
                target_url=twr_url,
                description=ghe_description,
                context="ci/ansbile-tower"
            )
            return "pass"
        elif ghe_action == 'changed_files':
            pulls = repo.get_pulls()
            pulls_numbers_list = []
            for pull in pulls:
                #TODO - This just gets the last pull, if multiple pulls are pending, you need to get the pull number, we have the data in the webhook GHE_ISSUE_ID.  #TODO
                pulls_numbers_list.append(pull.number)
            pull = repo.get_pull(pulls_numbers_list[0])
            filez = pull.get_files()
            return filez
    except Exception as e:
        print(e)
        return "err"




if __name__ == '__main__':

    # Update Github that we have started

    #TODO - We are repeating here a ton, we can narrow this down to a list of commands, run through the list, it's like this from testing.  also, make sure os.wait() is in the right spoot :shrug: #TODO
    githubcommand("status_update", "pending", TOWER_URL, "Ansible Tower Branch Check")

    # Setup Tower admin
    #TODO - Really just edit the  vim ~/.tower_cli.cfg file with the configs, tower-cli get's weird, and i think tower-cli was replaced by another tool, research needed
    if os_command_check_for_error("tower-cli config username " + TOWER_ADMIN) != "err":
        pass
    else:
        githubcommand("status_update", "failure", TOWER_URL, "Tower-Cli Failure,username")
        exit()

    # Setup Tower password
    if os_command_check_for_error("tower-cli config password " + TOWER_PASSWORD) != "err":
        pass
    else:
        githubcommand("status_update", "failure", TOWER_URL, "Tower-Cli Failure,password")
        exit()
    # Setup Tower host
    if os_command_check_for_error("tower-cli config host " + TOWER_HOST) != "err":
        pass
    else:
        githubcommand("status_update", "failure", TOWER_URL, "Tower-Cli Failure,host")
        exit()

    # Update the Tower-Project with the SCM info of the branch to check
    if os_command_check_for_error("tower-cli project modify --scm-url '" + GHE_SSH_URL + "' --scm-branch '" + GHE_BRANCH + "' " + TOWER_PROJECT_ID) != "err":
        pass
    else:
        githubcommand("status_update", "failure", TOWER_URL, "Tower-Cli Failure,Update SCM")
        exit()


    # Tell the project to go ahead and pull down the changes made inside of tower
    try:
        TOWER_JOB_ID = os_command_return_lines("tower-cli project update " + TOWER_PROJECT_ID)
    except:
        githubcommand("status_update", "failure", TOWER_URL, "Tower-Cli Failure,Project Update")
        exit()
    finally:
        pass

    #Before we move forward, we need to check that this was actually completed. it seems we are rushing the process and are checking old files.
    #Ideally we would check the ansible-tower api to ensure the job has compelted.  but for now just wait 5 seconds.  future self #TODO

    time.sleep(5)

    # Update GitHub that we are ready to check playbooks ***NOW WITH A LINK THAT DOES SOMETHING CLICKABLE****
    githubcommand("status_update", "pending", TOWER_URL + "/#/jobs/project/" + TOWER_JOB_ID, "Testing Playbooks...")

    # Pull down the files that changed in this pull request and test them
    changed_file_list = (githubcommand("changed_files"))
    pass_test = True
    #TODO - We should check if the file is in /playbooks long before we start to parse it. #TODO
    for xFile in changed_file_list:
        check_it = False
        if xFile.status == "added":
            check_it = True
        if xFile.status == "modified":
            check_it = True

        if check_it:
          #Make sure this is just a playbook, dont' check all of the files
          if "playbooks/" in xFile.filename:
              #TODO - Add in a check here if this playbook exists in a job, as you are building it, this is all adhock, the end game is to have the playbook run to update whatever was just approved.  leave for future versions #TODO
            if ".yml" in xFile.filename:
                # Tell Github, this may be too much, but why not?
                githubcommand("status_update", "pending", TOWER_URL + "/#/jobs/project/" + TOWER_JOB_ID,
                              "Testing " + xFile.filename)

                if os_command_check_for_error("ansible-playbook --check " + TOWER_PROJECT_LOCATION + xFile.filename) != "err":
                    githubcommand("status_update", "pending", TOWER_URL + "/#/jobs/project/" + TOWER_JOB_ID,
                                  xFile.filename + " Seems ok")
                else:
                    pass_test = False
                    githubcommand("status_update", "failure", TOWER_URL + "/#/jobs/project/" + TOWER_JOB_ID,
                                  xFile.filename + " Failed Ansible Test")
                    exit()

    # Finally Check if eveyrthing passed and get out of this situation
    if pass_test == False:
        #We should have had a failed test.  the last file that failed will be the message the users see, so they know what to look at.
        pass
    else:
        githubcommand("status_update", "success", TOWER_URL + "/#/jobs/project/" + TOWER_JOB_ID,
                      "ci/ansbile-tower tests passed.")

     #TODO - Really need to log out to something, so we can feed back the URL of the ansible run on a failure, will help speed up the debug, all you get now is that something is worng with the playbook, no details :shrug:
*******************************************

chmod 700 /usr/local/bin/ansible-tower-ci.py


Might have noticed a .netpass.ini file, this is where we are storing passwords for now. 

vim /etc/python/.netpass.ini

**netpass.ini**
[GitHub]
token = {{ GET_HUB_PERSONAL_TOKEN }}
url = {{ YOUR_GITHUB_REPO_API_LOCATION_exp_https://github.com/api/v3 }}
repo = {{ NAME_OF_YOUR_REPO_exp____bobby/ansible-tower }}
[Tower]
host = https://127.0.0.1
admin = admin
password = {{ TOWER_ADMIN_PASSWORD }}
url = {{ URL_OF_WHERE_TOWER_IS_RUNNING_exp_https://mytowerlb.com }}
project_id = {{ WHAT_PROJECt_ID_IS_YOUR_GITHUB_CI_CD_RUNNING_ON_IN_TOWER_exp_10_}}
project_location = {{ /var/lib/awx/projects/_10__bobby_githib_ci_cd_project/ <-Where is your Project running from }}

*********************************


chmod 0644 /etc/systemd/system/webhook.service
systemctl restart webhook
systemctl enable webhook 

--FireallD :: firewall-cmd --zone=public --add-port=9000/tcp ::

:::IF YOU ARE ADDING ANOTHER SERVER TO A GROUP OF WEBHOOKS, YOU ARE DONE, IF NOT, YOU NEED TO SETUP GITHUB


Browse to your github repo, goto the settings.
	exp: https://giThub.com/bobysproject/settings

Click on Hooks, Add webhook


Notes:  If you want, you can test by putting in the ip address of the machine, but once it's all said and done, you should be using a loadbalancer!
        Remember GitHub was the hook name, if your name is differnt, change it in webhook.conf as the ID, in these instructions we used GitHub

Playload URL:  https://{{LOAD_BALANCER_IP}}:9000/hooks/GitHub
Content Type:  application/json
Secret:  {{ SECRET_YOU_PUT_IN_WEBHOOK.CONF }}

--Let me select individual events
[*] - Pull Requests

[*] - Active

Update/Create/Green button, click it




--Things--


on your app server where the webhook is running, tail -f /var/log/messages  this will help you see if the playbook has any errors, also if webhook is having issues communicatin.  there is the were the output of your python file go


fork your github repo, make a change to a .yml file in the folder playbooks, assuming you have a folder called playbooks, if not, review the python code, we are reviewing for a specific folder, you know what, go ahead and just create a folder
called playbooks, just end this all. 

Inside of playbooks, create test.yml, put --- or anything in this, and put in a pull request to the main repo, if you see on the GitHub gui, that if it's all working, it is testing your playbook.

back on the hook/settings page, all webhooks you have done are sitting there, instead of constantly pushing a new file to test, just recreate the hook request, they are at the bottom if you edit the webhook.
if you click on it, it will redeliver, also you can see the whole payload, that helps with troublehsooting, if you need more details from the pull request, you can snag it here and place it in the webhook.conf



tower-cli config file example for when you remove the tower-cli commands

vim ~/.tower_cli.cfg

**.tower_cli.cfg**
# User options (set with `tower-cli config`; stored in ~/.tower_cli.cfg).
username: admin
password: {{ ANSIBLE_TOWER_ADMIN_PASSWORD }} 
host: http://127.0.0.1

# Defaults.
verbose: False
verify_ssl: False
use_token: False
oauth_token:
format: human
description_on: False
certificate:
color: True
insecure: False

************************************












