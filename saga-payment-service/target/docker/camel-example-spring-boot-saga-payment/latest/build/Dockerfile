FROM registry.access.redhat.com/ubi9/openjdk-17:latest
ENV JAVA_MAIN_CLASS=org.springframework.boot.loader.launch.JarLauncher JAVA_APP_DIR=/deployments
LABEL org.label-schema.description="Payment Service" org.label-schema.version=4.19.0-SNAPSHOT org.label-schema.schema-version=1.0 org.label-schema.build-date=2026-03-12 org.label-schema.name="Camel SB Examples :: Saga :: Payment Service" org.label-schema.vcs-ref=d94fd8369e51fa33319ea006856cf1ec2be70446 org.label-schema.usage=scpexe://people.apache.org/www/camel.apache.org/maven//examples/camel-example-spring-boot-saga/camel-example-spring-boot-saga-payment org.label-schema.url=https://www.apache.org/ org.label-schema.vendor="The Apache Software Foundation" org.label-schema.vcs-url=https://github.com/apache/camel-spring-boot-examples.git
EXPOSE 8081 8778 9779
COPY /deployments /deployments/
WORKDIR /deployments
