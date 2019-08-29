Role Name
=========

The purpose of this ansible role is to automate the Spark 2.3.2 with Jupyter Notebook for analytics lab.

Requirements
------------

Openshift 4.1
Ansible 2.x

Role Variables
--------------

The deployment is static then not require variables, only important thing is the OCP namespace that is: my-analytics

Dependencies
------------

No dependencies 

Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

- name: Starting build images
  hosts: localhost
  pre_tasks:
    - name: Builds messages
      debug:
        msg: "Builds in progress"

  roles:
  - ocp_spark_role

  post_tasks:
    - name: Post message
      debug:
        msg: "Builds completed"

License
-------

BSD

Author Information
------------------

EMEA SSA TEAM
