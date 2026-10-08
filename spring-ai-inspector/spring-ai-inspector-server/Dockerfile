FROM eclipse-temurin:25-jre

WORKDIR /app
COPY build/libs/spring-ai-inspector-server.jar inspector.jar

ENV SERVER_ADDRESS=0.0.0.0
EXPOSE 9001
USER 10001:10001
ENTRYPOINT ["java", "-jar", "/app/inspector.jar"]
