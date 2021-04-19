### doesn't work as shell command.
```
ansible multi -b -a "tail /var/log/messages"
```
### other way around 
## shows how many anisble commands have been run on each server
```
ansible multi -b -m shell -a "tail /var/log/messages | grep ansible-command | wc -l"
```
### cron
```
ansible multi -b -m cron -a "name='daily-cron-all-servers' hour=4 job='/path/to/daily-script.sh'"
ansible multi -b -m cron -a "name='daily-cron-all-servers' state=absent"
```
### GIT
```
ansible app -b -m package -a "name=git state=present" ##if u get unknown host_key , add this accept_hostkey=yes
ansible app -b -m git -a "repo=git://example.com/path/to/repo.git dest=/opt/myapp update=yes version=1.2.4"
ansible app -b -a "/opt/myapp/update.sh"
```
##### Ansible play books 
### touch playbok.yml
```
---
 - hosts: all

 tasks:
 - name: Install Apache.
   command: yum install --quiet -y httpd httpd-devel
 - name: Copy configuration files.
 command: >
 cp httpd.conf /etc/httpd/conf/httpd.conf
 - command: >
 	cp httpd-vhosts.conf /etc/httpd/conf/httpd-vhosts.conf
 - name: Start Apache and configure it to run at boot.
   command: service httpd start
 - command: chkconfig httpd on
 ```
 #### run ansible-playbook playbook.yml
 
 #### other way around 
 ```
 ---
 - hosts: all
   become: yes

 tasks:
 - name: Install Apache.
	 yum:
		 name:
		 - httpd
		 - httpd-devel
 		 state: present

 - name: Copy configuration files.
   copy:
     src: "{{ item.src }}"
     dest: "{{ item.dest }}"
     owner: root
     group: root
     mode: 0644
  with_items:
    - src: httpd.conf
      dest: /etc/httpd/conf/httpd.conf
    - src: httpd-vhosts.conf
      dest: /etc/httpd/conf/httpd-vhosts.conf

   - name: Make sure Apache is started now and at boot.
     service:
        name: httpd
       state: started
        enabled: yes
```
###  limiting 
```
ansible-playbook playbook.yml --limit webservers
ansible-playbook playbook.yml --list-hosts
ansible-playbook playbook.yml --become --become-user=janedoe --ask-become-pass
ansible-playbook playbook.yml --user=johndoe
##-i inventory path , --verbose or -vvvv, --extra-vars=VARS  , key=value format
##--check to check , --connection=TYPE , -c tye of conecction = ssh , 
```
#### installing npm 
```
---
 - hosts: all
   become: yes

   vars:
     node_apps_location: /usr/local/opt/node

   tasks:
   - name: Install EPEL repo.
     yum: name=epel-release state=present

   - name: Import Remi GPG key.
     rpm_key:
         key: "https://rpms.remirepo.net/RPM-GPG-KEY-remi"
         state: present

   - name: Install Remi repo.
	 yum:
     name: "https://rpms.remirepo.net/enterprise/remi-release-7.rpm"
     state: present

   - name: Ensure firewalld is stopped (since this is for testing).
     service: name=firewalld state=stopped

   - name: Install Node.js and npm.
	 yum: name=npm state=present enablerepo=epel

   - name: Install Forever (to run our Node.js app).
       npm: name=forever global=yes state=present
       
   - name: Ensure Node.js app folder exists.
       file: "path={{ node_apps_location }} state=directory" 
       
   - name: Copy example Node.js app to server.
       copy: "src=app dest={{ node_apps_location }}"
       
   - name: Install app dependencies defined in package.json.
        npm: path={{ node_apps_location }}/app
        
    - name: Check list of running Node.js apps.
	   command: forever list
	   register: forever_list
	   changed_when: false
	   
    - name: Start example Node.js app.
         command: "forever start {{ node_apps_location }}/app/app.js"
         when: "forever_list.stdout.find(node_apps_location + '/app/app.js') == -1"
```         
 ### ansible-playbook playbook.yml --extra-vars="node_apps_location=/usr/local/opt/node"
         
#### LAMP SETUP , create to files named playbook.yml , vars.yml     
 ```        
---
 - hosts: all
   become: yes

  vars_files:
  - vars.yml
  ### pre tasks ensures apt cache is valide for 1 hour or so we can use pre_tasks and post_tasks
	pre_tasks:
     - name: Update apt cache if needed.
       apt: update_cache=yes cache_valid_time=3600
      
  ### handlers are special kind of tasks , notify the tasks in that goup , handler will be called. 
  ### notfied at the end of the play      
      handlers:
      - name: restart apache
          service: name=apache2 state=restarted
      tasks:
      - name: Get software for apt repository management.
        apt:
		 name:
			- python-apt
  		     - python-pycurl
	      state: present

     - name: Add ondrej repository for later versions of PHP.
       apt_repository: repo='ppa:ondrej/php' update_cache=yes

     - name: "Install Apache, MySQL, PHP, and other dependencies."
        apt:
 		name:
		 - acl
		 - git
  		 - curl
		 - unzip
 		 - sendmail
		 - apache2
         - php7.1-common
		 - php7.1-cli
		 - php7.1-dev
		 - php7.1-gd
		 - php7.1-curl
		 - php7.1-json
		 - php7.1-opcache
		 - php7.1-xml
		 - php7.1-mbstring
		 - php7.1-pdo
		 - php7.1-mysql
		 - php-apcu
		 - libpcre3-dev
		 - libapache2-mod-php7.1
		 - python-mysqldb
		 - mysql-server
		state: present
		
	 - name: Disable the firewall (since this is for local dev only).
		service: name=ufw state=stopped
		
	 - name: "Start Apache, MySQL, and PHP."
		service: "name={{ item }} state=started enabled=yes"
		with_items:
		       - apache2
		       - mysql
	 - name: Enable Apache rewrite module (required for Drupal).
		apache2_module: name=rewrite state=present
		notify: restart apache
		
	 - name: Add Apache virtualhost for Drupal 8 development.
	    template:
			src: "templates/drupal.test.conf.j2"
			dest: "/etc/apache2/sites-available/{{ domain }}.test.conf"
			owner: root
			group: root
			mode: 0644
	   notify: restart apache
		
	 - name: Symlink Drupal virtualhost to sites-enabled.	
       file:
		src: "/etc/apache2/sites-available/{{ domain }}.test.conf"
		dest: "/etc/apache2/sites-enabled/{{ domain }}.test.conf"
		state: link
	   notify: restart apache   
	  
	 - name: Remove default virtualhost file.
	   file:
		  path: "/etc/apache2/sites-enabled/000-default.conf"
		  state: absent
	   notify: restart apache 
	#### configuring php 
     - name: Adjust OpCache memory setting.
        lineinfile:
			dest: "/etc/php/7.1/apache2/conf.d/10-opcache.ini"
			regexp: "^opcache.memory_consumption"
			line: "opcache.memory_consumption = 96"
			state: present
        notify: restart apache
	 - name: Create a MySQL database for Drupal.
       mysql_db: "db={{ domain }} state=present"
       
     - name: Create a MySQL user for Drupal.
        mysql_user:
			name: "{{ domain }}"
			password: "1234"
			priv: "{{ domain }}.*:ALL"
			host: localhost
			state: present
	 - name: Download Composer installer.
	   get_url:
			url: https://getcomposer.org/installer
			dest: /tmp/composer-installer.php
			mode: 0755
  
     - name: Run Composer installer.
	   command: >
			php composer-installer.php
			chdir=/tmp
			creates=/usr/local/bin/composer
  
     - name: Move Composer into globally-accessible location.
       command: >
		mv /tmp/composer.phar /usr/local/bin/composer
		creates=/usr/local/bin/composer
	 - name: Check out drush 8.x branch.
       git:
		repo: https://github.com/drush-ops/drush.git
		version: 8.x
		dest: /opt/drush
      
     - name: Install Drush dependencies with Composer.
       command: >
			/usr/local/bin/composer install
			chdir=/opt/drush
			creates=/opt/drush/vendor/autoload.php
			
     - name: Create drush bin symlink.
       file:
			src: /opt/drush/drush
			dest: /usr/local/bin/drush
			state: link  
		
	 - name: Check out Drupal Core to the Apache docroot.
        git:
			repo: https://git.drupal.org/project/drupal.git
			version: "{{ drupal_core_version }}"
			dest: "{{ drupal_core_path }}"
			register: git_checkout
        
     - name: Ensure Drupal codebase is owned by www-data.
          file:
            path: "{{ drupal_core_path }}"
            owner: www-data
            group: www-data
            recurse: true
          when: git_checkout.changed | bool
	 - name: Install Drupal dependencies with Composer.
         command: >
			/usr/local/bin/composer install
			chdir={{ drupal_core_path }}
			creates={{ drupal_core_path }}/vendor/autoload.php
         become_user: www-data
        
     - name: Install Drupal.
         command: >
			drush si -y --site-name="{{ drupal_site_name }}"
			--account-name=admin
			--account-pass=admin
			--db-url=mysql://{{ domain }}:1234@localhost/{{ domain }}
			--root={{ drupal_core_path }}
			creates={{ drupal_core_path }}/sites/default/settings.php
         notify: restart apache
         become_user: www-data
```
	  
### INSIDE THE VARSFILLE vars.yml
```
drupal_site_name: "Drupal Test"
drupal_core_version: 	  
drupal_site_name:
domain:	  
### Read fromm page number 86
###### Create new one 
```
