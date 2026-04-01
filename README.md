# 5873302-acm-observability
Ansible playbooks for IaC Automation of ACM on OCP with Observability

Make sure you set your `ansible_python_interpreter` for the local-host where the ansible-playbooks will be run from in `inventory.ini`.
Ideally, it should point to a python bin in your venv that has ansible, the ansible-k8s module, and other dependencies installed. 
To install dependencies, create and source a venv, and then run `pip(3) install -r requirements.txt` to install dependencies.
