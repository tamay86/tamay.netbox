Ansible Role: tamay.netbox
=========

This role installs the IP address management (IPAM) and data center infrastructure management (DCIM) tool [netbox](https://github.com/netbox-community/netbox).

After installation, you have to manually create a user to login.

    python3 /opt/netbox/netbox/manage.py createsuperuser


Requirements
------------

Netbox requires a postgres database and a webserver. 

- tamay.selinux
- tamay.httpd
- tamay.postgresql

Role Variables
--------------

List of all variables, including default values.

    # Version of Netbox to install
    # See https://github.com/netbox-community/netbox/releases
    netbox_version: '2.6.1'

Defines which version of netbox should be installed. It will be extracted to the folder ```/opt/netbox-2.6.1```. It will then be symlinked to ```/opt/netbox```. With this you can have multiple versions downloaded and just change the symlink to activate another version (useful for updating).
    
    # Configuration for netbox/netbox/configuration.py
    netbox_config:
      allowed_hosts:
        - "{{ ansible_fqdn }}"
        - "{{ ansible_default_ipv4.address }}"
      # Database connection
      database_name: netbox
      database_user: netbox
      database_password: netbox
      database_host: localhost
      database_port: 5432

This dictionary sets the configuration inside the file ```configuration.py```. See https://netbox.readthedocs.io/en/stable/configuration/required-settings/ for more information.

Dependencies
------------

None.

Example Playbook
----------------

- Configures selinux
- Installs webserver from tamay.httpd role
- Installs postgres from tamay.postgresql role
- Installs netbox with default httpd SSL certificates

Playbook:

    ---
    
    - hosts: all
      remote_user: vagrant
      become: yes
    
      vars:
        selinux_boolean:
          - name: httpd_can_network_connect
            state: yes
            persistent: yes
    
        httpd_modules:
          - mod_wsgi
        httpd_vhosts:
          - servername: "{{ ansible_fqdn }}"
            options: |
              Redirect permanent / https://{{ ansible_fqdn }}/
        httpd_vhosts_ssl:
          - servername: "{{ ansible_fqdn }}"
            errorlog: /var/log/httpd/{{ ansible_fqdn }}-error.log
            customlog: /var/log/httpd/{{ ansible_fqdn }}-access.log
            options: |
                ProxyPreserveHost On
                Alias /static /opt/netbox/netbox/static
                WSGIPassAuthorization on
                <Directory /opt/netbox/netbox/static>
                    Options Indexes FollowSymLinks MultiViews
                    AllowOverride None
                    Require all granted
                </Directory>
                <Location /static>
                    ProxyPass !
                </Location>
                ProxyPass / http://127.0.0.1:8001/
                ProxyPassReverse / http://127.0.0.1:8001/
                RequestHeader set X-Forwarded-Proto "https"
            ssl_protocol: "{{ httpd_ssl_protocol }}"
            ssl_cipher: "{{ httpd_ssl_cipher }}"
            ssl_honor_cipher: "{{ httpd_ssl_honor_cipher }}"
            ssl_certificate: "{{ httpd_ssl_certificate }}"
            ssl_key: "{{ httpd_ssl_key }}"
            ssl_chain: "{{ httpd_ssl_chain }}"
            ssl_ca: "{{ httpd_ssl_ca }}"
            ssl_hsts: "{{ httpd_ssl_hsts }}"
    
        postgresql_version: '96'
        postgresql_hba:
          - name: '"local" is for Unix domain socket connections only'
            type: local
            database: all
            user: all
            method: trust
          - name: 'IPv4 local connections:'
            type: host
            database: all
            user: all
            address: 127.0.0.1/32
            method: md5
          - name: 'IPv6 local connections:'
            type: host
            database: all
            user: all
            address: '::1/128'
            method: md5
        postgresql_databases:
          - name: "{{ netbox_config.database_name }}"
        postgresql_users:
          - name: "{{ netbox_config.database_user }}"
            password: "{{ netbox_config.database_password }}"
            db: "{{ netbox_config.database_name }}"
            priv: ALL
    
        netbox_version: '2.6.1'
        netbox_config:
          allowed_hosts:
            - "{{ ansible_fqdn }}"
          database_name: netbox
          database_user: netbox
          database_password: netboxpassword
          database_host: localhost
          database_port: 5432
    
    
      roles:
        - tamay.selinux
        - tamay.httpd
        - tamay.postgresql
        - tamay.netbox


License
-------

MIT

Author Information
------------------

tamay.mueller@gmail.com
