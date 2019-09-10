FROM maven:3.5-jdk-7 AS build

#~~~~~~~SNYK Variable~~~~~~~~~~~~
# Declare Snyktoken as a build-arg
ARG snyk_auth_token
# Set the SNYK_TOKEN environment variable
ENV SNYK_TOKEN=${snyk_auth_token}
#~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

# set the working directory
WORKDIR /usr/src/app

# copy just the pom.xml
COPY ./app/pom.xml /usr/src/app/pom.xml

# just install the dependencies for caching
RUN mvn -T 4C -T 4 dependency:go-offline

# copy the application code
COPY ./app /usr/src/app

# package the application
RUN mvn package -T 4C -T 2 -Dmaven.test.skip=true

#~~~~~~~SNYK test~~~~~~~~~~~~
# download, configure and run snyk. Break build if vulns present, post results to snyk.io
RUN curl -Lo ./snyk "https://github.com/snyk/snyk/releases/download/v1.210.0/snyk-linux"
RUN chmod -R +x ./snyk
#Auth set through environment variable
RUN ./snyk test --severity-threshold=medium
RUN ./snyk monitor
#~~~~~~~~~~~~~~~~~~~~~~~~~~~~


# create our Wildfly based application server
FROM jboss/wildfly:11.0.0.Final AS application

# install postgresql support
RUN mkdir -p $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
COPY ./postgresql $JBOSS_HOME/modules/system/layers/base/org/postgresql/main
RUN /bin/sh -c '$JBOSS_HOME/bin/standalone.sh &' \
  && sleep 10 \
  && $JBOSS_HOME/bin/jboss-cli.sh --connect --command="/subsystem=datasources/jdbc-driver=postgresql:add(driver-name=postgresql,driver-module-name=org.postgresql, driver-class-name=org.postgresql.Driver)" \
  && $JBOSS_HOME/bin/jboss-cli.sh --connect --command=:shutdown \
  && rm -rf $JBOSS_HOME/standalone/configuration/standalone_xml_history/ \
  && rm -rf $JBOSS_HOME/standalone/log/*

# copy and deploy the war file from build layer to application layer
COPY --from=build /usr/src/app/target/applicationPetstore.war /opt/jboss/wildfly/standalone/deployments/applicationPetstore.war

# copy our configuration
COPY ./standalone.xml /opt/jboss/wildfly/standalone/configuration/standalone.xml

# install nc for entrypoint script and copy the entrypoint script
USER root
RUN yum install nc -y
USER jboss
COPY ./docker-entrypoint.sh /opt/jboss/docker-entrypoint.sh

# expose the application port and the management port
EXPOSE 8080 9990

# run the application
ENTRYPOINT [ "/opt/jboss/docker-entrypoint.sh" ]
CMD [ "-b", "0.0.0.0", "-bmanagement", "0.0.0.0" ]
