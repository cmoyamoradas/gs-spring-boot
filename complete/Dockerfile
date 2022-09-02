FROM openjdk:8-jdk-alpine
ARG JAR_FILE
COPY ${JAR_FILE} spring-boot-complete.jar
ENTRYPOINT ["java","-jar","/spring-boot-complete.jar"]
