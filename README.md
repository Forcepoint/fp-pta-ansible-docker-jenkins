# docker-jenkins

Setup the host to configure and start Jenkins and associated docker containers. HTTPS is enabled by default.

This role is designed to manage the Jenkins overall version and all plugin versions, plus whatever other application
settings you wish. This is done by building our own docker container and installing the desired
plugins into it at build time. The base Jenkins image provides an easy means to do this. The configuration
of the Jenkins application beyond that is controlled via the Configuration as Code plugin.

## First Run

By design, the first run Jenkins wizard is not disabled. While you could hypothetically set the login
auth and other settings via the Configuration as Code plugin, the vanilla setup does not do this.
When you go through the first run wizard, do not install any plugins. That has and should have been
already handled by the docker image build (see Plugins section below).

## Configuration as Code

See https://github.com/jenkinsci/configuration-as-code-plugin/blob/master/README.md for information about
what can be managed with the Configuration as Code plugin.

To provide your own configuration file, use the docker_jenkins_casc_path variable. 
See the example playbook for how to override this variable.

While CASC can accept a URL for the yaml file, it is recommended to provide the file directly with Ansible.

**WARNING**: Use of the CASC file to manage nodes/slaves is not currently supported with these ansible
roles. Use the jenkins-node-jnlp and jenkins-node-ssh roles instead.

## Plugins

The plugins that get installed are controlled via a text file during the docker image build process.

**WARNING**: Plugins installed via the UI will be retained as default. Further configuration of that same
plugin via the text file will not work as Jenkins is designed to honor plugins installed via the UI first.
Be careful as this could lead to configuration drift.

It is recommended for Production Jenkins instances to
upgrade plugins via the text file. For a non-Production instance, upgrading via the UI for testing is
permissable, but still not recommeded.

It is also recommeded to provide your own text file which explicitly tracks all the plugin versions.
If you inspect this role's plugins.txt file, you'll notice that only a small subset of plugins are shown
and they are all set to the latest version. This is just for convenience of maintaining the role.
All the other plugins that appear in an actual installation are secondary dependent plugins (they appear
greyed out in the Plugin Installed list in the UI and you cannot uncheck them).
Please do not use the approach of only specifying the primary plugins with latest as their version 
in a Production Jenkins instance. Instead explicitly list all plugins and their versions to be installed 
in your own file. See the example playbook for how to override the docker_jenkins_plugins variable.

Determining the name of a plugin to list in the file can be a little tricky. 
See https://github.com/jenkinsci/docker#script-usage for more information. One method I found to determine a plugins
name for the text file is to search for the plugin on https://plugins.jenkins.io. The name of the plugins page
in the URL will be the desired spelling.

If you are able to use a test Jenkins instance to install all plugins as desired, running this in the Script Console
will yield the desired format and spelling.

    Jenkins.instance.pluginManager.plugins.each{
      plugin -> 
        println ("${plugin.getShortName()}:${plugin.getVersion()}")
    }

## Intermediate Certificate

Be aware that you may need to add an intermediate cert to your chain when you get your certificate. 
I had to do that for such a cert which was from GoDaddy. 

1. You have to figure out what the intermediate cert is that you need. 
   You can do this by running an openssl command to get the common name of the cert, then searching the web for it. 
   You'll end up at the certificate issuer's webpage as they maintain the list of intermediate certs. 
   If you're being extra careful, verify the checksum's match of the cert you 'think' is the one on their website 
   versus the one referenced by the cert you have. I won't go into how to do that. Download the pem file.
1. This [page](https://www.digicert.com/ssl-support/pem-ssl-creation.htm) told me how to combine the pem 
   files together - just open them in a text editor and just append the contents of the intermediate cert's 
   pem file onto the certicate's pem file. Once combined, you're good to go.

## Disaster Recovery

If you have utilized the backup-local role to make backups, follow these steps to recover from backups of the data...

1. For the sake of this example, let's assume...
   1. Your docker data (var: docker_gitlab_root_data_dir) is located `/home/service/data`.
   1. You backups (var: backup_local_destination) are all located `/home/service/data/backups`.
1. Run ansible to setup the application for you. As long as ansible completes, you're good to proceed.
1. Stop the running docker container.

        docker container stop nginx
        docker container stop jenkins
       
1. Delete the docker container. This is to ensure the volumes are released.

        docker container rm nginx
        docker container rm jenkins
       
1. Delete the docker data.

        sudo rm -rf /home/service/data/jenkins_home

1. Extract the backups.

        sudo tar -C / -xvf /home/service/data/backups/jenkins_home_DATETIME.tar

1. Run ansible to start the application.

## Requirements

Run the role docker-host on the host. The following files are required to be present. 
Ensure they're vaulted before committing them to source code control.

* files/<docker_jenkins_dns>.key
* files/<docker_jenkins_dns>.pem

## Role Variables

### REQUIRED
* docker_jenkins_dns: The DNS name for the Jenkins server. This DNS entry must already exist.

### OPTIONAL
* docker_jenkins_version_jenkins: The version of Jenkins to install. The default should always be an LTS release.
* docker_jenkins_version_nginx: The version of NGINX to install. The default is always going to be a stable release.
* docker_jenkins_timezone: Default timezone to use for Jenkins in the container. Otherwise UTC is used.
* docker_jenkins_root_data_dir: The directory to store the data in relevant to running the docker containers.
* docker_jenkins_update_center: The Jenkins Update Center for getting plugins from. 
  * Do not include the file update-center.json as part of the URL. The plugin install process will handle it. 
    If you do add that, it may cause errors during the docker image build process.
  * Setting this value will not set the update center in the UI. You must do that manually. This only affects
    the update center used when plugins are installed during the docker image build.
  * If you setup a mirrored repository to https://updates.jenkins.io in Artifactory, after the first time
    you download update-center.json, the file will no longer update. This is because in Artifactory,
    remote repositories aren't mirrored repositories. This means that once a file is retrieved, it won't check
    to see if the file changed. This isn't true for files with special meta-data, but for update-center.json,
    it is true. The best/simplest method I've found for triggering a new file is to delete the file from
    the cache repository. Then next attempt to obtain the file will cause Artifactory to get a current copy.
    I simply create a Jenkins job that runs nightly which does this delete.
* docker_jenkins_footer_url: The company URL to display at the bottom of the page.
* docker_jenkins_plugins: The path to a text file describing what plugins to install and their versions.
* docker_jenkins_casc_path: The path to the Configuration as Code yaml file to use.
* docker_jenkins_java_keystore: The path to the java keystore. The path for Java 8 is different for Java 11.
* docker_jenkins_certs_to_trust: A list of certificates to add into 
  Jenkins's java keystore and whether they are remote or not.
  Useful if you're using a private CA for Jenkins' web certificates. 
  Also useful if Jenkins needs to interact with other web applications whose
  certificates aren't in the java keystore.
  * If you use Active Directory authentication for your security mechanism, you'll run into
    this issue with Java versions beyond 8 if you do not add your AD cert to the keystore.
    In all honesty, I would avoid the Active Directory security realm if you use MS AD.
    Use LDAP instead against MS AD. We found it to be much faster and more reliable.
    https://issues.jenkins-ci.org/browse/JENKINS-52374

## Dependencies

None

## Example Playbook

    - hosts: servers
      vars:
        docker_jenkins_dns: jenkins.COMPANY.com
        docker_jenkins_plugins: "{{ playbook_dir }}/files/plugins.txt"
        docker_jenkins_casc_path: "{{ playbook_dir }}/templates/jenkins.yaml"
        docker_jenkins_certs_to_trust:
          # You baked your private CA certificate into the base image, use remote_src yes.
          - { path: '/etc/pki/ca-trust/custom/private_ca.pem', remote_src: yes }
          # The AD certificate. It's not included in the Java truststore by default.
          # May make more sense to get it from the playbook than to bake it into the base image. Use remote_src no.
          - { path: 'files/ms_ad.pem', remote_src: no }
      roles:
         - role: docker-jenkins

## License

BSD-3-Clause

## Author Information

Jeremy Cornett <jeremy.cornett@forcepoint.com>
