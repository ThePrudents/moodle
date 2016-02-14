

# Moodle installation on Ubuntu 14.04 in BlueMix cloud environment with ansible

## Introduction

Moodle is an open source Course Management Platform that allows you
to create online learning sites. The name stands for
Modular Object-Oriented Dynamic Learning Environment
and it has number of features allowing for an efficient online
learning experience that can scale from a small number of students to
hundreds.

For example, you could introduce assignment submission, quizzes, FAQ, scoring,
instant messages, discussion boards, etc. But
As it is a modular software, it can be extended with plugins to add extra functionality.


Moodle typically runs on LAMP (Linux, Apache, MySQL and PHP) but can
also be used with other webservers, like nginx and even on Windows under IIS.


Modularity is both pros and cons - usually, if you are the teacher, you have your own preferred set of themes and plugins.
This makes initial Moodle installation a real pain for non-technical users.

Fortunately, deployment automation tool called ansible will save you time on installation

# Background


## Challenges to address
  * Describe desired Moodle components
  * Optionally install MySQL (you might use external database)
  * Optionally install LAMP (perhaps you upgrade Moodle itself).
  * Optionally install Moodle's plugin and themes of your choice
  * Install Moodle core components

Let's go step by step.



## Describing desired Moodle components

According to official installation tutorial found on https://docs.moodle.org/30/en/Step-by-step_Installation_Guide_for_Ubuntu
Let's list both package dependencies and needed php extensions.
<pre>
  pkg_dependencies:
    - git
    - curl
    - python-dev
    - libmysqlclient-dev
    - graphviz
    - aspell
    - clamav
    - unzip

  php_extensions:
    - php5-mysql
    - php5-intl
    - php5-xmlrpc
    - php5-pspell
    - php5-curl
    - php5-gd
    - php5-ldap
</pre>

For Moodle itself we need to specify preferred database access parameters,
domain you plan to assign for the Moodle instance, preferred location.
Also you might want to check what is the most recent stable Moodle version and
specify it as well.
<pre>
  moodle_db_type: 'mysqli' #  'pgsql', 'mariadb', 'mysqli', 'mssql', 'sqlsrv' or 'oci'
  moodle_db_host: '{{mysql_host}}'
  moodle_db_name: 'moodle'
  moodle_db_user: 'moodle'
  moodle_db_pass: 'moodle'
  moodle_app_domain: "moodle.dev"
  moodle_app_root: "/opt/moodle"
  moodle_app_wwwroot: "{{ moodle_app_root }}/moodle"
  moodle_app_datadir: "{{moodle_app_root}}/moodledata"
  moodle_app_plugindir: "{{moodle_app_root}}/downloadedplugins"
  # Check on site - at that moment 30 is recent stable
  moodle_git_version: "MOODLE_30_STABLE"
  moodle_artifact_baseurl: https://download.moodle.org/download.php/direct/stable30
  moodle_archive_version: "moodle-latest-30.tgz"

  moodle_user: "{{ansible_user_id}}"

  moodle_admin_user: "6NHkm*S!^W4w"

</pre>

Moodle can be installed either from git repository or via artifact downloading.
Git option is believed to be maintained more easily.

We support both ways here, you set the best for your case
<pre>
# Currently git is easiest , also supported: web,
  option_install_moodle: git
</pre>

## Optional MySQL installation

Althouth Moodle supports multiple databases, MySQL is the most typical and  known database on unix.
In order to install MySQl, you will need to specify desired mysql root credentials only.
<pre>
  mysql_host: "127.0.0.1"
  mysql_root_user: root
  mysql_root_password: SOMEROOTSECUREPASSWORD
</pre>

Moodle has it's own preferences to MySQL configuration. These are addressed my custom my.cnf template,
as you could check under that link: https://github.com/ThePrudents/moodle/blob/master/templates/mysql/my.cnf.j2

Self contained recipe to install MySQL might be found here: https://github.com/ThePrudents/moodle/blob/master/tasks/mysql.yml

## Optional LAMP installation
For LAMP we have ability to install apache either in worker or prefork mode.
This has impact on PHP: it will be installed either as PHP-FPM or via mod_php apache module.
Nowadays worker is preferable.
<pre>
  apache_mode: worker # use prefork or worker variables
  apache2_disable_default: true

  php_family: default # 5.4 | 5.5 | 5.6 | default
</pre>

You might want to check recipes to install Apache (https://github.com/ThePrudents/moodle/blob/master/tasks/apache.yml)
and PHP (https://github.com/ThePrudents/moodle/blob/master/tasks/php_apache.yml) for more details.

Note, as Moodle requires additional php extensions to be installed, we have introduced additional step,
 as can be checked here: https://github.com/ThePrudents/moodle/blob/master/tasks/php_additional_extensions.yml

## Optional custom Moodle's plugins and themes
Most of plugins and themes are installed by unzipping archieve to appropriate folders.
We could easily automate that process while you need just tell us your preferred plugins information:
<pre>
 moodle_plugins:
    - {
      name: auth_googleoauth2,
      desc: "Authentication: Google / Facebook / Github / Linkedin / DropBox / Windows / VK / Battle.net authentication",
      url: "https://moodle.org/plugins/download.php/9695/auth_googleoauth2_moodle30_2015110600.zip",
      dest: auth #/googleoauth2
      }
    - {
      name: mod_checklist,
      desc: "Activities: Checklist",
      url: "https://moodle.org/plugins/download.php/9703/mod_checklist_moodle30_2015110800.zip",
      dest: mod #/checklist
      }
    - {
      name: block_xp,
      desc: "Blocks: Level up!",
      url: "https://moodle.org/plugins/download.php/9400/block_xp_moodle30_2015092800.zip",
      dest: blocks #/block_xp
      }
    - {
      name: theme_essential,
      desc: "Themes: Essential",
      url: "https://moodle.org/plugins/download.php/10342/theme_essential_moodle30_2016010201.zip",
      dest: theme #/theme_essential
      }
</pre>

## Install Moodle core components
On this step we create Moodle's directories (source, data, plugins), perform
database configuration, configure Apache virtual host for this instance.

Steps details might be checked at this location:
https://github.com/ThePrudents/moodle/blob/master/tasks/moodle.yml

## Code in action

In order to be able to run provisioning, we would need ansible and prudentia. Both tools are pure python.
Ansible is devops toolkit, while prudentia provides some "synthetic sugar" to run ansible playbooks easily.
See more details on pypi https://pypi.python.org/pypi/prudentia
For ubuntu your typical steps to get tools are:
<pre>
sudo apt-get install git
sudo apt-get install python-pip
pip install -U pip
pip install -U ansible==1.9.4
pip install prudentia
</pre>

to reuse Moodle role, typical syntax is: (please note, that you may override any role parameter,
that makes recipe flexible enough)
<pre>
  roles:
     - {
         role: "pr-moodle",
         moodle_db_host: '127.0.0.1',
         moodle_db_name: 'moodle',
         moodle_db_user: 'moodle',
         moodle_db_pass: 'yoursupersecurepassword',
         moodle_app_domain: "yourdomainname.com"

       }
</pre>

Shell file to execute box provisioning
<pre>

# Static parameters
WORKSPACE=./
BOX_PLAYBOOK=$WORKSPACE/boxes/prod.yml
BOX_NAME=moodle_staging
BOX_ADDRESS=192.168.0.17
BOX_USER=youruser
BOX_PWD=yourpass

prudentia ssh <<EOF
unregister $BOX_NAME

register
$BOX_PLAYBOOK
$BOX_NAME
$BOX_ADDRESS
$BOX_USER
$BOX_PWD

provision $BOX_NAME
EOF
</pre>

