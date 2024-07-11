
---
- name: Install Odoo
  hosts: all
  become: yes
  vars:
    odoo_user: "odoo"
    odoo_home: "/{{ odoo_user }}"
    odoo_home_ext: "/{{ odoo_user }}/{{ odoo_user }}-server"
    install_wkhtmltopdf: True
    odoo_port: "8069"
    odoo_version: "16.0"
    is_enterprise: False
    install_postgresql_fourteen: True
    install_nginx: False
    odoo_superadmin: "admin"
    generate_random_password: True
    odoo_config: "{{ odoo_user }}-server"
    website_name: "_"
    longpolling_port: "8072"
    enable_ssl: True
    admin_email: "odoo@example.com"
  tasks:
    - name: Ensure apt cache is updated
      apt:
        update_cache: yes

    - name: Install required repositories
      apt_repository:
        repo: "{{ item }}"
      with_items:
        - "ppa:deadsnakes/ppa"
        - "deb http://mirrors.kernel.org/ubuntu/ xenial main"
    
    - name: Upgrade all packages to the latest version
      apt:
        upgrade: dist
        update_cache: yes

    - name: Install dependencies
      apt:
        name: "{{ item }}"
        state: present
      loop:
        - libpq-dev
        - python3
        - python3-pip
        - git
        - python3-cffi
        - build-essential
        - wget
        - python3-dev
        - python3-venv
        - python3-wheel
        - libxslt-dev
        - libzip-dev
        - libldap2-dev
        - libsasl2-dev
        - python3-setuptools
        - node-less
        - libpng-dev
        - libjpeg-dev
        - gdebi
        - curl

    - name: Install PostgreSQL
      apt:
        name: "postgresql-14"
        state: present
      when: install_postgresql_fourteen

    - name: Create Odoo PostgreSQL user
      become_user: postgres
      postgresql_user:
        name: "{{ odoo_user }}"
        password: ""

    - name: Install python packages
      pip:
        requirements: "https://github.com/odoo/odoo/raw/{{ odoo_version }}/requirements.txt"

    - name: Clone Odoo
      git:
        repo: "https://www.github.com/odoo/odoo"
        dest: "{{ odoo_home_ext }}"
        version: "{{ odoo_version }}"
        depth: 1

    - name: Create Odoo system user
      user:
        name: "{{ odoo_user }}"
        home: "{{ odoo_home }}"
        shell: /bin/bash
        system: yes

    - name: Create log directory
      file:
        path: "/var/log/{{ odoo_user }}"
        state: directory
        owner: "{{ odoo_user }}"
        group: "{{ odoo_user }}"

    - name: Create custom module directory
      file:
        path: "{{ odoo_home }}/custom/addons"
        state: directory
        owner: "{{ odoo_user }}"
        group: "{{ odoo_user }}"
        recurse: yes

    - name: Set permissions on home folder
      file:
        path: "{{ odoo_home }}/*"
        owner: "{{ odoo_user }}"
        group: "{{ odoo_user }}"
        recurse: yes

    - name: Generate random admin password
      set_fact:
        odoo_superadmin: "{{ lookup('password', '/dev/null length=16 chars=ascii_letters') }}"
      when: generate_random_password

    - name: Create server config file
      copy:
        dest: "/etc/{{ odoo_config }}.conf"
        content: |
          [options]
          admin_passwd = {{ odoo_superadmin }}
          http_port = {{ odoo_port }}
          logfile = /var/log/{{ odoo_user }}/{{ odoo_config }}.log
          addons_path = {{ odoo_home_ext }}/addons,{{ odoo_home }}/custom/addons

    - name: Create startup file
      copy:
        dest: "{{ odoo_home_ext }}/start.sh"
        content: |
          #!/bin/sh
          sudo -u {{ odoo_user }} {{ odoo_home_ext }}/odoo-bin --config=/etc/{{ odoo_config }}.conf
      mode: '0755'

    - name: Create init file
      copy:
        dest: "/etc/init.d/{{ odoo_config }}"
        content: |
          #!/bin/sh
          ### BEGIN INIT INFO
          # Provides: {{ odoo_config }}
          # Required-Start: $remote_fs $syslog
          # Required-Stop: $remote_fs $syslog
          # Should-Start: $network
          # Should-Stop: $network
          # Default-Start: 2 3 4 5
          # Default-Stop: 0 1 6
          # Short-Description: Enterprise Business Applications
          # Description: ODOO Business Applications
          ### END INIT INFO
          PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin
          DAEMON={{ odoo_home_ext }}/odoo-bin
          NAME={{ odoo_config }}
          DESC={{ odoo_config }}
          USER={{ odoo_user }}
          CONFIGFILE="/etc/{{ odoo_config }}.conf"
          PIDFILE=/var/run/$NAME.pid
          DAEMON_OPTS="-c $CONFIGFILE"
          [ -x $DAEMON ] || exit 0
          [ -f $CONFIGFILE ] || exit 0
          checkpid() {
            [ -f $PIDFILE ] || return 1
            pid=`cat $PIDFILE`
            [ -d /proc/$pid ] && return 0
            return 1
          }
          case "$1" in
          start)
            echo -n "Starting $DESC: "
            start-stop-daemon --start --quiet --pidfile $PIDFILE \
            --chuid $USER --background --make-pidfile \
            --exec $DAEMON -- $DAEMON_OPTS
            echo "$NAME."
            ;;
          stop)
            echo -n "Stopping $DESC: "
            start-stop-daemon --stop --quiet --pidfile $PIDFILE \
            --oknodo
            echo "$NAME."
            ;;
          restart|force-reload)
            echo -n "Restarting $DESC: "
            start-stop-daemon --stop --quiet --pidfile $PIDFILE \
            --oknodo
            sleep 1
            start-stop-daemon --start --quiet --pidfile $PIDFILE \
            --chuid $USER --background --make-pidfile \
            --exec $DAEMON -- $DAEMON_OPTS
            echo "$NAME."
            ;;
          *)
            N=/etc/init.d/$NAME
            echo "Usage: $NAME {start|stop|restart|force-reload}" >&2
            exit 1
            ;;
          esac
          exit 0
      mode: '0755'

    - name: Enable Odoo service
      command: update-rc.d {{ odoo_config }} defaults

    - name: Start Odoo service
      service:
        name: "{{ odoo_config }}"
        state: started
        enabled: yes

    - name: Install Wkhtmltopdf
      apt:
        name: wkhtmltopdf
        state: present
      when: install_wkhtmltopdf

    - name: Install Nginx
      apt:
        name: nginx
        state: present
      when: install_nginx

    - name: Configure Nginx
      copy:
        dest: "/etc/nginx/sites-available/{{ website_name }}"
        content: |
          server {
            listen 80;

            server_name {{ website_name }};

            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Real-IP $remote_addr;
            add_header X-Frame-Options "SAMEORIGIN";
            add_header X-XSS-Protection "1; mode=block";
            proxy_set_header X-Client-IP $remote_addr;
            proxy_set_header HTTP_X_FORWARDED_HOST $remote_addr;

            access_log /var/log/nginx/{{ odoo_user }}-access.log;
            error_log /var/log/nginx/{{ odoo_user }}-error.log;

            proxy_buffers 16 64k;
            proxy_buffer_size 128k;

            proxy_read_timeout 900s;
            proxy_connect_timeout 900s;
            proxy_send_timeout 900s;

            proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;

            types {
              text/less less;
              text/scss scss;
            }

            gzip on;
            gzip_min_length 1100;
            gzip_buffers 4 32k;
            gzip_types text/css text/scss text/less text/plain text/xml application/xml application/json application/javascript application/pdf image/svg+xml;
            gzip_vary on;
            gzip_proxied any;
            gzip_disable "MSIE [1-6]\.";

            location / {
              proxy_pass http://127.0.0.1:{{ odoo_port }};
              proxy_redirect off;
            }

            location /longpolling {
              proxy_pass http://127.0.0.1:{{ longpolling_port }};
            }

            location ~* /web/static/ {
              proxy_cache_valid 200 90m;
              proxy_buffering on;
              expires 864000;
              proxy_pass http://127.0.0.1:{{ odoo_port }};
            }

            location ~* /[0-9a-zA-Z_]*/static/ {
              proxy_cache_valid 200 90m;
              proxy_buffering on;
              expires 864000;
              proxy_pass http://127.0.0.1:{{ odoo_port }};
            }

            include /etc/nginx/snippets/ssl-{{ website_name }}.conf;
            include /etc/nginx/snippets/ssl-params.conf;
          }
      when: install_nginx

    - name: Enable Nginx site
      file:
        src: "/etc/nginx/sites-available/{{ website_name }}"
        dest: "/etc/nginx/sites-enabled/{{ website_name }}"
        state: link
      when: install_nginx

    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
      when: install_nginx







[odoo]
192.168.1.10 ansible_user=your_ssh_user

ansible-playbook -i hosts install_odoo.yml --ask-become-pass



