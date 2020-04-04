![Alt text](images/Ansible-logo1.png?raw=true)

# ansible local fact example
Scaling any config management tool can be a daunting task. What starts as some code to fix or do one task, quickly escalates to many tasks on many servers. Only now you have conditional situations adding further complexity to your code base.

This is where local facts come in. Local facts are great. Multiple server builds can be based on them and they can serve the purpose of an ENC (external node classifier) which can save you effort and time as you scale out your ansible implementation.

This demo gives an overview of how they can give you back control as you scale your code.

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

## Run the ansible playbook
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

## The fact file
This is the file containing the local facts. Make a note of how the file is setup. Name encased on [] at the top. Key value pairs are listed below separated by an "=".

<details>
 <summary>local_facts.yml:</summary>
  <p>

````
[localfacts]
role= webserver
environment= production
owner= systems
appport= 9090
````

</p></details>

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

</p></details>

## Display the facts in a playbook run:
In this playbook we display our local facts in the playbook. The link to the variables is established in our group/all variables file. You don;t need to use this file, and could just use the variable names, but it's always good practice to group variables together so you can update everything in one place.

<details>
 <summary>show_facts.yml:</summary>
  <p>

````
---
- name: print app port
  debug: msg="your application port is {{ list_appport }}"
- name: print environment
  debug: msg="your environment is {{ list_environment }}"
- name: print role
  debug: msg="your role is {{ list_role }}"
````

</p></details>

## Group variables
See the group variables below.

<details>
 <summary>group/all:</summary>
  <p>

````
---

list_appport: "{{ ansible_local.local.localfacts.appport | default('9999') }}"
list_environment:  "{{ ansible_local.local.localfacts.environment }}"
list_role:  "{{ ansible_local.local.localfacts.role }}"
````

The link breaks down to this (ansible_local.local.localfacts.appport):

````
ansible_local = This is the default starting place for all local variables in ansible
local = This is the filename holding the facts. The file name is local.fact
localfacts = This is the headed of the local.fact file and is located at the top of the screen in square brackets [localfacts]
appport = This is the variable name and will give the value of the Key --> value in the local.fact file
````
The | default('9999') section lets us add a default if there is no local_fact for that particular variable.

</p></details>


## The template_example.yml
This playbook creates a file containing the local variables we just created. This proves that local variables can be used within templates.

<details>
 <summary>template_example.yml:</summary>
  <p>

````
---

- template:
    src: ~/ansible_local_facts/templates/ansible_local_vars.j2
    dest: /tmp/local_ansible_variables
````

</p></details>

## The template jinja2 file
We can use the variables in our templates. This template shows how we get the values out of the local facts file and into a seperate file which could be a configuration file on a server.

<details>
 <summary>templates/ansible_local_vars.j2:</summary>
  <p>

````
These are the ansible variables from the playbook:
you can use them in a template like this:

your application port is {{ list_appport }}
your environment is {{ list_environment }}
your role is {{ list_role }}
````

</p></details>

## The local file output
When we run ansible, the file will get created with our facts inside. This is the output of the file:

<details>
 <summary>/tmp/local_ansible_variables:</summary>
  <p>

````
These are the ansible variables from the playbook:
you can use them in a template like this:

your application port is 9090
your environment is production
your role is webserver
````

</p></details>

## Presedence and Order
You can test out which variable will be used by Ansible using this example:

'#' out the appport line in files/local.fact

Then run ansible again. The default value will be picked up in the absences of the local fact.

Now run the same code using "--extra-vars" and change the port. Extra-vars is the king. It's top of the presedence rules.
````
$ sudo ansible-playbook -i localhost run.yml --extra-vars list_appport='1010'
````

The output will show the new port value.


Feel free to Star my repo if you found it helpful.
