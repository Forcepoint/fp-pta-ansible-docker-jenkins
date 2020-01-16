FROM jenkins/jenkins:{{ docker_jenkins_version_jenkins }}

USER root

{% if docker_jenkins_certs_to_trust is defined %}
# Add in any certificates to the system keystore.
COPY certs/* /usr/local/share/ca-certificates/
RUN update-ca-certificates

# Add them into the java keystore as well.
COPY certs/* /home/certs/
{% for item in docker_jenkins_certs_to_trust %}
RUN $JAVA_HOME/bin/keytool -import -storepass changeit -noprompt -trustcacerts -alias {{ (item.path | basename | splitext)[0] }} -file /home/certs/{{ item.path | basename }} -keystore {{ docker_jenkins_java_keystore }}
{% endfor %}
{% endif %}

# Install plugins.
ENV JENKINS_UC=https://slcartifactory.us.fdo/artifactory/jenkins-updates/stable
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN /usr/local/bin/install-plugins.sh < /usr/share/jenkins/ref/plugins.txt

# Drop back to the regular jenkins user - good practice.
# https://github.com/jenkinsci/docker/blob/master/README.md#installing-more-tools
USER jenkins