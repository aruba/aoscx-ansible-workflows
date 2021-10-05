# aoscx-ansible-workflows
This repository contains ready-to-use workflows using the [AOS-CX Ansible Collection](https://galaxy.ansible.com/arubanetworks/aoscx). For detailed guides on using Ansible with AOS-CX visit us on our [Aruba Developer Hub](https://developer.arubanetworks.com/aruba-aoscx/docs/ansible-getting-started).


Requirements
------------

* Python 3 or later
* Ansible 2.9.0 or later
  * Refer to [Ansible's documentation](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) for installation steps
* Minimum supported AOS-CX firmware version 10.04
* Enable REST on your AOS-CX device with the following commands:
    ```
    switch(config)# https-server rest access-mode read-write
    switch(config)# https-server vrf mgmt
    ```
* Enable SSH on your AOS-CX device with the following command:    
    ```
    switch(config)# ssh server vrf mgmt
    ```


Executing Workflows
------------
1. Ensure you've completed all the requirements stated above.
2. Clone this repository onto your Ansible Control Machine:
  ```
  git clone https://github.com/aruba/aoscx-ansible-workflows.git
  ```
3. Modify the `inventory.yml` file to match the IP address and login credentials of your AOS-CX switch:
  * `ansible_host`: IP address of switch in `A.B.C.D` format. For IPv6 hosts use a string and enclose in square brackets E.G. `'[2001::1]'`.
  * `ansible_user`: Username for switch in `plaintext` format  
  * `ansible_password`: Password for switch in `plaintext` format
4. Run the desired workflow using the `ansible-playbook` command and providing the `inventory.yml` file:
  ```
  ansible-playbook configure_vlans.yml -i inventory.yml
  ```
  
Contribution
-------
At Aruba Networks we're dedicated to ensuring the quality of our products, so if you find any
issues at all please open an issue on our [Github](https://github.com/aruba/aoscx-ansible-workflows) and we'll be sure to respond promptly!  

For more contribution opportunities follow our guidelines outlined in our [CONTRIBUTING.md](https://github.com/aruba/aoscx-ansible-workflows/blob/master/CONTRIBUTING.md)

License
-------

Apache 2.0