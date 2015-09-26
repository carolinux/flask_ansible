# Installing Flask with Ansible on CentOS 7

In this guide, we will be deploying a simple Flask application on a CentOS 7 droplet. We will use the provisioning tool Ansible to carry out this task, instead of running commands one by one. While building our deployment script, we will get the chance to explore a wide array of Ansible's features and also some best practices.

## Prerequisites
Before starting on this guide, you should have a non-root user configured on your server. The user will be responsible for setting up and keeping the website running (and it's not a good idea for a website to be running as root). This user needs to have sudo privileges so that it can perform administrative functions, such as installing packages. To learn how to set this up, follow our [initial server setup guide](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04) . 

You also need to have ansible installed on your local machine, that is, the machine you use to ssh into your droplet. You can find installation instructions for all platforms in the [official Ansible documentation](http://docs.ansible.com/ansible/intro_installation.html).

## Why Ansible

Using a provisioning framework has the advantage of letting us do a one-command deployment where we don't need to interact with the terminal. Ansible in particular is built around the concept of *indempotency* - the capability to run the same deployment script again and again and to have the certainty that the system will always reach the desired state safely, without unnecessarily redoing work that has already been done or throwing errors when a command is executed twice. 

That means, for example, that commands such as "create a certain directory" are better expressed as "make sure that a certain directory is present in the system". Ansible works by letting you define "playbooks" which contain the commands that you need executed. The playbooks are written in [YAML](http://yaml.org/). With that in mind, we can get started writing our very own playbook to install Flask and deploy a sample application.

## Building the Playbook

First, we need to create a plain text file named flask.yml

### Defining variables
At the top of the file, we need to add the following lines:
```
- hosts: all

  vars:
    LOCAL_PROJECT_HOME: "./hello"
    PROJECT_HOME: "~/my_test_home"
    MAIN_PY: "hello.py" # main file to launch website
    PORT: 5000 # port of the website
```
The first line signifies that we want to run this command on all the hosts that we are given. The deployment section will show how launch a playbook together with a host file to define the hosts for Ansible.

  The vars section lists all the variables we're going to be using throught the playbook.

### Flask Hello World

We're going to be copying the local directory (LOCAL_PROJECT_HOME) to the PROJECT_HOME directory on the droplet. It contains the standard [hello world](http://flask.pocoo.org/docs/0.10/quickstart/) Flask project. The differences are that the hostname of the app is "0.0.0.0", so as to make it accessible externally and that it's possible to configure the port the website runs on via a command line argument. This is what hello.py looks like:

```python
import sys

from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello World!'

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=sys.argv[1])

```

After defining the top section, we merely need to list several task items for Ansible to execute. 

### Copying the code 

```
  - name: Create remote directory 
    file: path={{PROJECT_HOME}} state=directory

  - name: Copy project code to website server
    copy: src={{item}} dest={{PROJECT_HOME}}
    with_fileglob:
     - "{{LOCAL_PROJECT_HOME}}/*"

```
The tasks of a playbook are defined in YAML as a list of dictionaries and executed from top to bottom. If we have several hosts, then each task is tried for each host before moving on to the next one. Each task is defined as a dictionary that can have several keys, such as "name" or "sudo" which signify the name of the task and whether it requires sudo privileges (the default is no). 

The only thing necessary to define a task is to have a line that defines a module to be called and specify some arguments. Modules are wrappers around lower level functionality such as "shell" (for running things on the shell), "file" (for file operations) or "copy"(a wrapper around scp). Ansible ships with several [core modules](http://docs.ansible.com/ansible/modules_core.html) that can cover most deployment tasks.

The first task asserts that the project home directory is present in the droplet, using the [file module](http://docs.ansible.com/ansible/file_module.html). Internally, Ansible will check if the directory exists and create it if it doesn't, otherwise it will do nothing.

 The second task has a slightly more complex definition - by using *with_fileglob*, it loops over all the files in the local directory and copies them over to the project directory in the droplet. *{{item}}* is the default name for an element resulting from a generator such as with_fileglob in Ansible. If they already exist in the remote directory with the exact same contents, they will not be copied (whereas scp always transfers everything). Conveniences like this are one of the reasons it's a good idea to use the existing modules for each task, instead of the plain shell module. 

Some alternatives to do the task of provisioning the code to the remote machine would be to use the synchronize module (based on rsync) to keep directories in sync or even the git module to check out code from a repository on the remote server.

### Installing necessary libraries
```
  - name: Install extra packages (to be able to install pip)
    yum: name=epel-release state=latest
    sudo: yes

  - name: Install lsof (for better process management)
    yum: name=lsof state=latest
    sudo: yes

  - name: Install pip
    yum: name=python-pip state=latest
    sudo: yes

  - name: Install virtualenv
    pip: name=virtualenv
    sudo: yes

  - name: Create virtualenv for project
    shell: virtualenv "{{ PROJECT_HOME }}/venv"
           creates="{{ PROJECT_HOME }}/venv/bin/activate"

  - name: Install Flask in the virtualenv 
    pip: name=flask virtualenv="{{ PROJECT_HOME }}/venv" 
```

The first task installs the packages necessary to install pip, Python's package manager. We also install lsof to use it later to easily terminate and restart our website process. These actions require sudo privileges, so we specify that as well. We use the yum module, that is, Ansible's module for the CentOS package manager. Let's say we tried to use the shell module instead, ie something like:
```
- name : Install epel
  shell: yum install epel-release
```
Then, Ansible would in fact get stuck because yum requires the user to confirm that they want to install the packages by using the keyboard. We could of course specify the -y flag to force the confirmation, but in general there's no need to use the shell when a well-documented ansible core module will do.



For the next task, we install virtualenv to separate the packages used for the website from the system Python. For such a simple example this is not really needed, but when deploying at a server with several python projects installed, separating each one in its own virtual environment makes life easier. 

The next task (creating a virtual environment for our project) is quite interesting from an indempotency point of view. We want to be able to know if the task has ran already, so that Ansible skips it the second time. By defining that this task creates a particular file when it has ran, Ansible knows to skip the whole task if the file already exists. This logic of course could be wrapped more neatly in a dedicated virtualenv module, but at the time of this writing, such a module doesn't exist in the core modules of Ansible.

Finally, we install the Flask library in the virtual environment we just created within our project directory.


### Launching the website
```
  - name: Get process id running the webserver
    shell: lsof -t -i:{{PORT}}
    sudo: yes
    ignore_errors: yes
    register: pid

  - name: Kill webserver if it's already running
    shell: kill {{pid.stdout}}
    when: pid.stdout != ""

  - name: Launch flask website via virtualenv
    shell: "source {{ PROJECT_HOME }}/venv/bin/activate; nohup python {{PROJECT_HOME}}/{{MAIN_PY}} 2>&1 >/dev/null &"

  - name: Wait for website port to become available
    wait_for: port={{PORT}} delay=1

```

The first command finds if any task is running in the defined port. By default, lsof is available only to root, so that's why we set sudo to yes. Alternatively, we could provide the full path to the lsof executable, but that's a bit less clean. We set this task to ignore errors so that if nothing is running in that port and lsof fails, we can still continue. By default, Ansible would stop the execution if a non-zero exit status is returned from a task.

 We register the output of the task in a variable called pid. The second task (which kills the process) is executed only if the standard output of the previous task is non-empty.

The third command activates the virtual environment and starts the webserver as a daemon. If the webserver was not started in this way, then the process would be killed as soon as Ansible disconnected from the remote server.

The last command waits for the choosen port to become available. The way it's defined, it would wait for ever, after initially waiting one second without polling. 




## Deploying

We need to create a *hosts.conf* file with the addresses of all the droplets we want to deploy at (one per line), for example:
```bash
95.85.38.227
95.85.38.226
```

Then we can launch our ansible script with the following command from our local machine (username is the user you want the webserver to run as):

```bash
ansible-playbook flask.yml -i hosts.conf -u username --ask-sudo-password
```

Ansible needs to create a temporary directory on the remote machine (by default under /tmp/) to keep track of the execution of the current playbook. Make sure this directory can be written to by the chosen user. You can change the default remote temporary directory by editing the file */etc/ansible/ansible.cfg* on your local machine, if you're using a Linux distribution.

It is possible to pass several flags to ansible-playbook. Two interesting ones are the -vvvv flag, which enables verbose output which is particularly useful to troubleshoot SSH issues and the --extra-vars command. If you want to override any of the vars defined in the playbook, you can do so in the command line by passing the flag like this:

```bash
--extra-vars "MAIN_PY=hello2.py PROJECT_HOME=/var/www/"
```

Launch the command, type in the sudo password when requested to do so and wait until it has finished. Then go to your droplet address at the chosen port (default:5000) and verify that you can see the hello world greeting.

If you rerun the command, you will see that several steps will be skipped or be marked as "unchanged".  If for some reason the script fails in the middle (let's say, a network error), when you rerun it you won't need to worry about commenting out what has already ran.

## Code
You can find all the code needed to experiment with the Flask installation via Ansible on my [github](https://www.github.com/carolinux/flask_ansible).



