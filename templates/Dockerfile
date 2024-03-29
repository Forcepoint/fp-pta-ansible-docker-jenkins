FROM jenkins/jenkins:{{ docker_jenkins_version_jenkins }}

USER root

{% if docker_jenkins_certs_to_trust is defined %}
# Add certs into the java keystore.
COPY certs/* /home/certs/
{% for item in docker_jenkins_certs_to_trust %}
RUN $JAVA_HOME/bin/keytool -import -storepass changeit -noprompt -trustcacerts -alias {{ (item.path | basename | splitext)[0] }} -file /home/certs/{{ item.path | basename }} -keystore {{ docker_jenkins_java_keystore }}
{% endfor %}
{% endif %}

# Add in any certificates to the system keystore. They must be CRT files.
RUN for file in /home/certs/*.pem; do mv "$file" "${file}.crt"; done
RUN cp /home/certs/*.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates

# Install plugins.
ENV JENKINS_UC={{ docker_jenkins_update_center }}
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

# Drop back to the regular jenkins user - good practice.
# https://github.com/jenkinsci/docker/blob/master/README.md#installing-more-tools
USER jenkins

{% if docker_jenkins_custom_plugins is defined %}
# Copy in custom plugins. https://github.com/jenkinsci/docker/#preinstalling-plugins
COPY plugins/* /usr/share/jenkins/ref/plugins/
{% endif %}