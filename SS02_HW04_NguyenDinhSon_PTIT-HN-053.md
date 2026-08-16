# Bài 4: Hybrid AI Runtime

## 1. `build.gradle`

```gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.3.2'
    id 'io.spring.dependency-management' version '1.1.6'
}

group = 'com.rlogistics'
version = '0.0.1-SNAPSHOT'

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

ext {
    set('springAiVersion', "1.0.0")
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.ai:spring-ai-starter-model-ollama'
    implementation 'org.springframework.ai:spring-ai-starter-model-openai'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.ai:spring-ai-bom:${springAiVersion}"
    }
}

tasks.named('test') {
    useJUnitPlatform()
}
```

## 2. `application-local.properties`

```properties
spring.ai.ollama.base-url=http://localhost:11434
spring.ai.ollama.chat.options.model=qwen2.5-coder:7b
```

## 3. `application-cloud.properties`

```properties
spring.ai.openai.api-key=${OPENROUTER_API_KEY}
spring.ai.openai.base-url=https://openrouter.ai/api/v1
spring.ai.openai.chat.options.model=google/gemini-2.5-flash
```

## 4. `application.properties`

```properties
spring.profiles.active=local
```

## 5. Chạy profile cloud từ CLI

Chạy bằng lệnh:

```bash
java -jar app.jar --spring.profiles.active=cloud
```

Hoặc nếu dùng Gradle:

```bash
./gradlew bootRun --args='--spring.profiles.active=cloud'
```

Khi bật `cloud`, Spring Boot sẽ tự lấy file `application-cloud.properties`.
