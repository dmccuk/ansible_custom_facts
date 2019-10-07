![Alt text](images/Ansible-logo1.png?raw=true)

# ansible local fact example

## Make sure you install or update to the latest version of Ansible.
In this example, I'll update the ansible version on ubuntu 16.04:

````
$ sudo apt-add-repository ppa:ansible/ansible
$ sudo apt-get update
$ sudo apt-get -y install ansible
$ sudo ansible --version
ansible 2.8.5
  config file = /home/vagrant/ansible_local_facts/ansible.cfg
  configured module search path = [u'/home/vagrant/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.12 (default, Aug 22 2019, 16:36:40) [GCC 5.4.0 20160609]

````
## Clone my repo
Take a local copy of the code onto your ubuntu server:

````
$ git clone https://github.com/dmccuk/ansible_local_facts.git
Cloning into 'ansible_local_facts'...
remote: Enumerating objects: 28, done.
remote: Counting objects: 100% (28/28), done.
remote: Compressing objects: 100% (16/16), done.
remote: Total 28 (delta 7), reused 27 (delta 6), pack-reused 0
Unpacking objects: 100% (28/28), done.
Checking connectivity... done.
````

## RUn the ansible playbook
Now you have the code, you should be able to run it with no updates. Run it first, then we can break down each playbook to see what's happening.

<details>
 <summary>Expand for the output:</summary>
  <p>

````
$ sudo ansible-playbook -i localhost run.yml
 [WARNING]: Unable to parse /home/vagrant/ansible_local_facts/localhost as an inventory source

 [WARNING]: No inventory was parsed, only implicit localhost is available

 [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not match 'all'


PLAY [localhost] *********************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************
ok: [localhost]

TASK [Create custom fact directory] **************************************************************************************************
ok: [localhost]

TASK [Insert custom fact file] *******************************************************************************************************
ok: [localhost]

TASK [local facts] *******************************************************************************************************************
ok: [localhost] => {
    "ansible_local": {
        "local": {
            "localfacts": {
                "appport": "9090",
                "environment": "production",
                "owner": "systems",
                "role": "webserver"
            }
        }
    }
}

TASK [reload facts] ******************************************************************************************************************
ok: [localhost]

TASK [print app port] ****************************************************************************************************************
ok: [localhost] => {
    "msg": "your application port is 9090"
}

TASK [print environment] *************************************************************************************************************
ok: [localhost] => {
    "msg": "your environment is production"
}

TASK [print role] ********************************************************************************************************************
ok: [localhost] => {
    "msg": "your role is webserver"
}

TASK [template] **********************************************************************************************************************
ok: [localhost]

PLAY RECAP ***************************************************************************************************************************
localhost                  : ok=9    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
````
</p></details>


## Localhost
I've run the playbook against my own VM for this demo but it will work against remote servers in the same way as the code is self contained.

The setup of the playbook is similar to a role in that we import the tasks into a main playbook (run.yml). This is a fairly standard way to run a collection of playbooks as a group against a host.

Check the syntax out:

````
---
- hosts: localhost
  connection: local

  tasks:
    - import_tasks: tasks/local_facts.yml
    - import_tasks: tasks/show_facts.yml
    - import_tasks: tasks/template_example.yml
````

if you create more playbooks in the tasks directory, make sure you add an "import_tasks" for them in the run.yml.

## local_facts.yml
This playbook creates our local facts file in the default ansible directory. It takes a local file called local.fact from the "files" directory and copies it to the default facts.d directory.

The important section here is the <b>notify</b>. You <b>MUST</b> add the notify and reload the facts using the setup function if you want to use the facts in other playbooks in this ansible run. If you miss it out, the playbook won't see the facts on the first run. On the second run the facts will then be visible.

<details>
 <summary>local_facts.yml:</summary>
  <p>
````
---
- name: "Create custom fact directory"
  file:
    path: "/etc/ansible/facts.d"
    state: "directory"
    mode: 0766

- name: "Insert custom fact file"
  copy:
    src: files/local.fact
    dest: /etc/ansible/facts.d/local.fact
    mode: 0644

- name: local facts
  debug: var=ansible_local
  notify:
  - reload facts

- name: reload facts
  setup: filter=ansible_local
````
<p><details>



