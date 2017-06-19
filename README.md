# Ansible Development by Example

- [Why?](#why)
- [What is this?](#what-is-this)
- [Environment setup](#environment-setup)
- [New module development](#new-module-development)
- [Local/direct module testing](#localdirect-module-testing)
- [Playbook module testing](#playbook-module-testing)
- [Debugging (local)](#debugging-local)
- [Unit testing](#unit-testing)
- Integration testing (coming soon)

### Why?

Ansible is an awesome tool for configuration management. It is also a highly utilized one, and there are so many ways to contribute as a community.

### What is this?

There is no doubt that Ansible is a complex tool, with lots of inner-workings, yet it is easy to work with as an end user. But on the other end of that, contributing to Ansible with code can sometimes be a daunting task.

This documentation is a way to show step-by-step how to develop Ansible modules, both new module development as well as bug fixes and debugging.

# Environment setup

1. Clone the Ansible repository: `$ git clone https://github.com/ansible/ansible.git`
1. Change directory into the repository root dir: `$ cd ansible`
1. Create a virtual environment: `$ python3 -m venv venv` (or for Python 2 `$ virtualenv venv`. Note, this requires you to install the virtualenv package: `$ pip install virtualenv`)
1. Activate the virtual environment: `$ . venv/bin/activate`
1. Install development requirements: `$ pip install -r requirements.txt`
1. Run the environment setup script for each new dev shell process: `$ . hacking/env-setup`

:zap: After the initial setup above, every time you are ready to start developing Ansible you should be able to just run the following from the root of the Ansible repo: `$ . venv/bin/activate && . hacking/env-setup`

:bulb: Starting new development now? Fixing a bug? Create a new branch: `$ git checkout -b my-new-branch`. If you are planning on contributing back to the main Ansible repostiry, fork the Ansible repository into your own GitHub account and developing against your new non-devel branch in your fork. When you believe you have a good working code change, submit a pull request to the Ansible repository.

# New module development

If you are creating a new module that doesn't exist, you would start working on a whole new file. Here is an example:

- Navigate to the directory that you want to develop your new module in. E.g. `$ cd lib/ansible/modules/cloud/azure/`
- Create your new module file: `$ touch my_new_test_module.py`
- Paste this simple into the new module file: (explanation in comments)
```python
#!/usr/bin/python

ANSIBLE_METADATA = {
    'metadata_version': '1.0',
    'status': ['preview'],
    'supported_by': 'curated'
}

DOCUMENTATION = '''
---
module: my_sample_module

short_description: This is my sample module

version_added: "2.4"

description:
    - "This is my longer description explaining my sample module"

options:
    name:
        description:
            - This is the message to send to the sample module
        required: true
    new:
        description:
            - Control to demo if the result of this module is changed or not
        required: false

extends_documentation_fragment
    - azure

author:
    - Your Name (@yourhandle)
'''

EXAMPLES = '''
# Pass in a message
- name: Test with a message
  my_new_test_module:
    name: hello world

# pass in a message and have changed true
- name: Test with a message and changed output
  my_new_test_module:
    name: hello world
    new: true

# fail the module
- name: Test failure of the module
  my_new_test_module:
    name: fail me
'''

RETURN = '''
original_message:
    description: The original name param that was passed in
    type: str
message:
    description: The output message that the sample module generates
'''

from ansible.module_utils.basic import AnsibleModule

def run_module():
    # define the available arguments/parameters that a user can pass to
    # the module
    module_args = dict(
        name=dict(type='str', required=True),
        new=dict(type='bool', required=False, default=False)
    )

    # seed the result dict in the object
    # we primarily care about changed and state
    # change is if this module effectively modified the target
    # state will include any data that you want your module to pass back
    # for consumption, for example, in a subsequent task
    result = dict(
        changed=False,
        original_message=''
        message=''
    )

    # the AnsibleModule object will be our abstraction working with Ansible
    # this includes instantiation, a couple of common attr would be the
    # args/params passed to the execution, as well as if the module
    # supports check mode
    module = AnsibleModule(
        argument_spec=module_args,
        supports_check_mode=True
    )

    # if the user is working with this module in only check mode we do not
    # want to make any changes to the environment, just return the current
    # state with no modifications
    if module.check_mode:
        return result

    # manipulate or modify the state as needed (this is going to be the
    # part where your module will do what it needs to do)
    result['original_message'] = module.params['name']
    result['message'] = 'goodbye'

    # use whatever logic you need to determine whether or not this module
    # made any modifications to your target
    if module.params['new']:
        result['changed'] = True

    # during the execution of the module, if there is an exception or a
    # conditional state that effectively causes a failure, run
    # AnsibleModule.fail_json() to pass in the message and the result
    if module.params['name'] == 'fail me':
        module.fail_json(msg='You requested this to fail', **result)

    # in the event of a successful module execution, you will want to
    # simple AnsibleModule.exit_json(), passing the key/value results
    module.exit_json(**result)

def main():
    run_module()

if __name__ == '__main__':
    main()
```

# Local/direct module testing

You may want to test the module on the local machine without targeting a remote host. This is a great way to quickly and easily debug a module that can run locally.

- Create an arguments file with the following content: (explanation below)
```json
{
  "ANSIBLE_MODULE_ARGS": {
    "name": "hello",
    "new": true
  }
}
```
- Ensure that you can directly execute your new module: `$ chmod 755 ./my_new_test_module.py`
- Run your test module locally and directly: `$ ./my_new_testmodule.py /tmp/args.json`

This should be working output that resembles something like the following:

```
{"changed": true, "state": {"original_message": "hello", "new_message": "goodbye"}, "invocation": {"module_args": {"name": "hello", "new": true}}}
```

:bulb: The arguments file is just a basic json config file that you can use to pass the module your parameters to run the module it

# Playbook module testing

If you want to test your new module, you can now consume it with an Ansible playbook.

- Create a playbook in any directory: `$ touch testmod.yml`
- Add the following to the new playbook file
```yaml
---
- name: test my new module
  connection: local
  hosts: localhost

  tasks:
    - name: run the new module
      my_new_test_module:
        name: 'hello'
        new: true
      register: testout

    - name: dump test output
      debug:
        msg: '{{ testout }}'
```
- Run the playbook and analyze the output: `$ ansible-playbook ./testmod.yml`

# Debugging (local)

If you want to break into a module and step through with the debugger, locally running the module you can do:

1. Set a breakpoint in the module: `import pdb; pdb.set_trace()`
1. Run the module on the local machine: `$ python -m pdb ./my_new_test_module.py ./args.json`

# Unit testing

Unit tests for modules will be appropriately located in `./test/units/modules`. You must first setup your testing environment. In my case, I'm using Python 3.5.

- Install the requirements (outside of your virtual environment): `$ pip3 install -r ./test/runner/requirements/units.txt`
- To run all tests do the following: `$ ansible-test units --python 3.5` (you must run `. hacking/env-setup` prior to this)

:bulb: Ansible uses pytest for unit testing

To run pytest against a single test module, you can do the following (provide the path to the test module appropriately):

```
$ pytest -r a --cov=. --cov-report=html --fulltrace --color yes test/units/modules/.../test_my_new_test_module.py
```
